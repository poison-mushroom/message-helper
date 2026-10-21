# Message Helper macOS 实现方案

日期：2026-10-21。对应 [需求背景与产品需求](requirements.md)。本文供用户将仓库拉取至 Mac 后开发使用，目前仓库尚无应用源码，以下目录与命令是待实现方案，不代表已经编译或验证。

## 1. 技术路线与前置条件

使用 Swift 5.9+，初始构建基线 macOS 13+，采用 Swift Package 管理纯逻辑库、系统适配库和桌面可执行程序。界面使用 SwiftUI＋AppKit，跨应用读写使用 Accessibility，发送按钮蒙版使用透明 NSPanel／NSWindow，快捷键接管候选为 CGEventTap。先不引入第三方运行时依赖。

这些是实施选择，不是兼容性结论。实际最低系统、架构和微信支持范围以用户 Mac 的验证结果为准。没有可靠的会话识别和草稿读取，就不能实现真实发送代理。

Mac 准备工作：

```bash
git clone https://github.com/poison-mushroom/message-helper.git
cd message-helper
xcode-select -p
xcrun swift --version
sw_vers
uname -m
```

已有检出直接 `git pull --ff-only`。若未安装开发工具，执行 `xcode-select --install` 并按系统提示完成安装；安装成功与否由后续命令验证。当前 Linux 云环境没有 Swift／Xcode，未执行 Mac 编译或宿主测试。

## 2. 首先做只读能力探测

实现 `Tools/AXProbe.swift`，检查用户是否已授予辅助功能权限，并枚举目标进程的 AX 元素角色、支持属性名、可写状态和动作名。默认不得输出 AXValue、窗口标题或聊天正文。

在自己的测试会话验证以下能力，结果记录至 `docs/validation/macos-wechat.md`：

- 可精确读取编辑框原文，包括中文、emoji、换行及链接。
- 可区分会话、窗口和编辑框实例，关闭重开窗口不会复用旧授权。
- 可取得发送按钮范围并正确转换到屏幕坐标。
- 可在确认原文未变时修改草稿，并发现写入失败。
- 可区分发送快捷键、换行及中文 IME 候选确认；不能可靠区分时报告不支持。
- 能否通过 AX 动作发送；如果只能注入键鼠事件，额外验证不会递归拦截、穿透或双发。

后续执行命令为 `swift Tools/AXProbe.swift`，工具尚待编写。若探测失败，记录具体属性缺失和应用版本，停止当前自动发送路径。不能因为可显示蒙版就声称发送保护可用。

## 3. 模块及目录

```text
Package.swift
Sources/
  GuardCore/
    Models.swift             # 目标身份、快照、问题和单次授权
    GuardCoordinator.swift   # 请求合并、失效、确认和发送协调
    Proofreader.swift        # 校对引擎协议与结构化结果
    Replacement.swift        # UTF-16 范围验证与安全替换
  GuardMac/
    WeChatAdapter.swift      # 微信 AX 识别、读写和发送动作
    PermissionMonitor.swift  # 权限、事件 tap 和可用性
    SendOverlay.swift        # 发送按钮覆盖窗口
    KeyboardInterceptor.swift
  MessageHelper/
    MessageHelperApp.swift
    StatusController.swift
    ReviewPanel.swift
Tests/
  GuardCoreTests/
  GuardMacTests/
Tools/AXProbe.swift
Resources/Info.plist
scripts/build-macos-app.sh
docs/validation/
```

GuardCore 不依赖 AppKit，系统 API 通过适配层注入。未来飞书和钉钉使用各自适配器，不把宿主差异塞进统一坐标规则。输入监听回调不执行网络、AX 遍历或复杂计算，只做预先缓存的目标判定并投递请求，避免事件 tap 超时。

## 4. 数据模型及接口约定

```swift
struct TargetIdentity: Equatable {
    let processID: Int32
    let launchToken: UUID
    let windowToken: UUID
    let conversationToken: String
    let editorToken: String
}

struct DraftSnapshot: Equatable {
    let target: TargetIdentity
    let revision: UUID
    let text: String
}

enum IssueCategory { case spelling, grammar, ambiguity, style }

struct Issue {
    let range: NSRange       // 当前快照的 UTF-16 范围
    let category: IssueCategory
    let explanation: String
    let replacements: [String]
}

protocol Proofreader {
    func check(_ snapshot: DraftSnapshot) async throws -> [Issue]
}
```

`TargetIdentity` 的令牌来自进程和 AX 元素生命周期管理，不能仅由窗口标题或屏幕位置生成。`revision` 是协调器跟踪的内容版本：同一草稿重新读取时保留版本，文本或目标变化才更新；不能每次 capture 都生成新版本导致所有复核失败。发送前始终重新读取并比较真实原文，不单靠版本号。

对用户可见的歧义候选允许多种意图，由用户选择。所有返回范围必须校验合法性、字符边界及原文一致性；重叠修改首版逐条应用并重新校验，不一次盲目批量替换。

适配器约定：

```swift
@MainActor protocol ChatAdapter {
    func capture() throws -> DraftSnapshot
    func replace(expected: DraftSnapshot, text: String) throws -> DraftSnapshot
    func send(expected: DraftSnapshot, permit: SendPermit) throws -> SendAttemptResult
}
```

`SendPermit` 是内部受控的一次性令牌，不提供公共构造器；由协调器授权并在适配器中统一消费。`SendAttemptResult` 至少区分 `actionInvoked` 与 `unknown`，不能把动作调用成功当作消息送达。未知结果不自动重试。

## 5. 发送状态机

```text
未保护 → 就绪 → 检查中 → 无问题 → 复核 → 单次发送 → 就绪
                    └→ 有问题 → 用户确认 ─┘
                    └→ 错误 → 重试／明确仍然发送／取消
任意阶段：目标变化、权限失效 → 取消请求及授权 → 未保护或重新就绪
```

所有状态变更在 MainActor 串行处理。每个活动请求绑定请求 ID、目标及草稿版本。引擎回调必须匹配当前请求；取消网络请求并不能替代结果失效检查。

实时检查采用可配置 debounce，初始实现可设 600ms 作为待测默认值，不作为性能承诺。IME 组合输入期间不检查中间文本。实时结果可缓存，但发送时必须重新捕获快照，仅当完整匹配且引擎配置未变时复用结果。

检查中重复点击不排队发送。点击“仍然发送”仅忽略校对问题，不能忽略会话变化或读取失败。切换会话取消原请求，不允许把旧意图迁移到新聊天。

提示面板会夺取焦点，因此持有原目标身份，不用“当前前台窗口”覆盖它；发送前恢复目标并重新验证会话。若无法确认恢复正确，取消发送。

## 6. 双入口实现细节

### 蒙版

覆盖目标发送按钮的透明窗口必须接收点击，并与目标窗口前台状态、位置、大小、屏幕、Spaces 和缩放变化同步。只靠轮询旧坐标不够；结合 AX 通知、应用切换及显示配置变化，并在每次操作前复核。

按钮位置不确定、目标被遮挡或切换应用时撤销旧蒙版并显示保护不可用，不能继续拦截其他软件。不能单靠 AX 框架位置推断按钮当前可见。

优先保持蒙版并通过已验证的 AX 发送动作触发宿主。若必须暂时隐藏窗口注入点击，需验证空档期间物理点击不会穿透造成双发；做不到则不支持该发送路径。

### 快捷键

CGEventTap 需要具备拦截能力的事件 tap，而非仅观察模式。按目标应用实际设置映射 Return、Command+Return 等组合，保留换行及 IME 候选确认。输入法状态无法确认时，不默认吞掉所有 Return，也不能宣称快捷键保护完整。

识别系统停用 event tap 的通知，立即撤销已保护状态；恢复监听成功并重新确认目标后才能重新启用。应用自己注入的发送事件使用可验证标记与活动令牌识别，不能采用“全局放行一段时间”的方案。

## 7. 校对与替换

三种语言场景共同保留，但不能由简单规则假装实现完整语义纠错。先定义稳定的 `Proofreader` 接口，以受控假引擎测试流程；测试引擎必须在界面标明“演示”，不得静默返回空问题并打开生产保护。

真实引擎接入前确定提供商、调用费用、文本传输范围、区域与保留策略。若使用云模型，客户端调用自有后端，服务密钥留在后端；身份认证、限流、超时与输出结构校验属于该接入阶段的正式设计，不能放一枚共享密钥进 App。

首版不依赖聊天历史来修正。建议文本中的姓名、金额、日期、链接、代码和否定语义需要专门评估，模型不能擅自改变事实。校对文本作为数据处理，不能执行其中的指令或工具调用。

替换前对快照做精确比较；通过 `Range(nsRange, in: text)` 并验证组合字符边界，拒绝非法、越界或破坏 emoji 的范围。采纳后重新检查，只有再次授权才发送。恢复原文也需要版本匹配。

## 8. 权限、运行与分发

按实际 API 请求“系统设置 → 隐私与安全性”中的辅助功能及系统要求的输入监控授权。拒绝、撤销、签名变化和重启后都需要重新检查，不把权限引导页当作已授权。

不默认请求屏幕录制权限；不绕过 Secure Input，不要求 root。未验证 Mac App Store 沙盒及审核兼容性，因此不承诺上架。第一阶段本地开发构建，后续直接分发需要 Developer ID 签名、公证及稳定 bundle ID；相关凭据不入库。

启动时没有权限、没有可支持客户端或没有真实引擎配置，均明确显示不可用原因，不自动发送。助手退出或崩溃后不具备阻断能力，这是产品边界，不以守护进程宣称解决绝对可靠性。

## 9. 实施顺序

| 阶段 | 产物 | 继续条件 |
| --- | --- | --- |
| 1 | AX 只读探测与真机兼容记录 | 能可靠识别会话、草稿和按钮；不能则调整路线 |
| 2 | GuardCore 状态机及测试 | 旧结果失效、单次授权、去重、错误恢复通过 |
| 3 | 系统适配及双入口 | 两种入口均不误拦其他应用或中文候选，无双发 |
| 4 | 菜单栏及确认界面 | 修正、取消、仍然发送及保护状态符合需求 |
| 5 | 真实校对引擎 | 三种场景有有效结果，隐私和费用配置明确 |
| 6 | 开发 App 打包及真机验收 | 支持矩阵完整，失败及未知结果不自动重试 |

逐任务接口、测试和执行步骤见 [详细开发计划](superpowers/plans/2026-10-21-macos-send-guard.md)。该计划覆盖保护框架，真实引擎仍需落实上述独立接入阶段。本文新增的实时检查与内容版本规则以本文为准。

源码实现后，在仓库根目录运行：

```bash
swift test
swift build -c release
bash scripts/build-macos-app.sh
```

目前这些目标和脚本尚不存在，以上不是立即可执行的安装说明。打包脚本须生成 `.app` 和 Info.plist，仅在 Darwin 运行；测试 runner 必须显示实际测试数量，零测试不算验证。

## 10. 验收矩阵与交付标准

- 文本：中文口语、正式中文、英语、中英混合、emoji、长文本、多行、链接、代码、姓名、金额和日期。
- 输入：鼠标、发送快捷键、换行、候选确认、粘贴、长按、双击、连续改稿。
- 目标：切换会话、切换应用、关闭并重开窗口、弹出校对面板、失焦和重新聚焦。
- 显示：移动窗口、调整大小、Retina／外接屏、切换缩放、断开显示器、Spaces、全屏和最小化。
- 故障：校对超时、断网、过期回调、AX 读取／写入失败、权限撤销、event tap 停用、应用退出。
- 发送：仅经用户许可向自己的测试会话发送；重复输入只触发一次，结果未知不重试，不能向第三方发送测试消息。
- 隐私：日志、错误报告和验证记录不含聊天原文，云服务启用有明确选择，密钥不在客户端或 Git 中。

每项记录系统、CPU 架构、微信版本、发送设置、显示环境、结果及复现步骤。模型质量和平台兼容性分开报告；单元测试通过不能替代 Mac 微信真机测试。

## 11. 当前交接状态

已完成需求及实现文档，未编写应用、未探测真实 AX 树、未配置模型服务、未编译、未签名公证。下一位开发者应从阶段 1 开始，先证明技术前提，再实现真实保护，不跳过前提直接写死微信界面。
