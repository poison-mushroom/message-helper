# macOS Send Guard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Delegation requires the user's explicit choice.

**Goal:** 在明确支持的 macOS 微信输入框，通过发送按钮蒙版和快捷键接管发送请求，在用户确认校对结果后只放行一次发送。

**Architecture:** 无第三方运行时依赖的 Swift Package，分离纯逻辑 GuardCore、macOS 系统适配 GuardMac 和 SwiftUI/AppKit 菜单栏程序。目标应用支持由真实 Accessibility 探测结果决定，任何假实现、模拟适配器或引擎空结果均不得让界面显示真实保护已启用。

**Tech Stack:** Swift 5.9+，macOS 13+ 为初始构建基线（实际支持以真机验证为准），SwiftUI、AppKit、ApplicationServices、XCTest；安装包签名及公证另需开发者证书。

**Spec:** `docs/research/2026-10-21-desktop-send-guard.md`，用户已于本轮确认交互方案。本文为首个 macOS 微信适配阶段的计划，不声称已完成竞品实测或校对模型选型。

## Global Constraints

- “以 macOS 为首发平台，暂以桌面微信为首个适配目标。”
- “不拦截语音、文件、图片或撤回操作。”
- “中文 IME 组合输入时，候选确认与换行必须保留，不能把所有 Enter 都当成发送。”
- “检查期间文字变化则丢弃旧结果；切换会话则取消待处理请求，禁止把上一会话的发送意图转移到新会话。”
- “发送结果不确定时不自动重试，避免重复消息。”
- “默认不保存消息正文或写入遥测。”
- Linux 当前无 Swift/Xcode；记录不能执行的命令，不将静态审阅当成编译或真机通过。
- 使用现有隔离检出，不另建 worktree。任何真实发送测试使用用户自己控制的测试会话，需有明确授权，不向第三方发送测试消息。

## Review Focus

- emoji、中文组合字符的 UTF-16 范围转换不得破坏替换：任务 2、4 测试覆盖。
- 关闭窗口后复用窗口标识，不得复用发送授权：任务 2、3 使用进程启动标识与窗口实例令牌测试。
- 显示器拔出及缩放切换，不得留在旧按钮坐标阻挡其他应用：任务 3 真机用例。
- 事件 tap 被停用或用户撤销权限，状态必须立即失效：任务 3 测试及真机用例。
- 校对面板抢走焦点后，不得把恢复焦点误认为用户选择了其他会话：任务 2、4 显式保存和复核目标身份测试。

## 文件职责

`Package.swift` 定义 GuardCore、GuardMac、MessageHelper 和对应测试；`Sources/GuardCore/Models.swift` 定义快照与问题；`GuardCoordinator.swift` 管理请求和授权；`Proofreader.swift` 定义检查协议；`Sources/GuardMac/WeChatAdapter.swift` 仅负责受支持微信界面；`PermissionMonitor.swift`、`SendOverlay.swift`、`KeyboardInterceptor.swift` 分别管理权限、覆盖窗口和事件 tap；`Sources/MessageHelper/` 负责应用生命周期与提示界面。`Tools/AXProbe.swift` 为本地只读探测工具；`docs/validation/` 保存不含私人文本的兼容性结果。

### Task 1: 先探测宿主能力，确定真实可支持范围

**Files:** Create `Tools/AXProbe.swift`, `docs/validation/macos-wechat.md`。

**Interfaces:** 探测器仅输出 role、可用属性名、是否可写、动作名及经过脱敏的窗口／进程标识；不得打印 AXValue、窗口标题或聊天正文。适配器后续使用 `TargetIdentity(processID: Int32, launchToken: UUID, windowToken: UUID, conversationToken: String, editorToken: String)`，其中会话令牌仅在内存使用。

- [ ] 编写只读探测器，默认不获取属性值；退出码 2 表示无权限，3 表示未找到可支持目标，0 仅表示探测成功。
- [ ] 在 Mac 执行 `swift Tools/AXProbe.swift`；拒绝授权时验证退出码 2，授予权限后记录系统版本、架构和微信版本。
- [ ] 在用户控制的测试会话确认草稿可精确读取、会话可可靠区分、输入框可写、按钮范围和发送动作可取得。测试会话切换、空草稿、长文本、emoji 和多行。仅记录通过／失败及元数据，不存消息文本。
- [ ] 若不能可靠识别草稿或会话，停止真实发送路线并报告具体失败项；不得以固定坐标或 OCR 替代身份验证。纯逻辑任务 2 可独立推进。
- [ ] 提交本任务文件；兼容性文档明确未测试项目，不预填通过。

### Task 2: 实现可测试的发送协调器

**Files:** Create `Package.swift`, `Sources/GuardCore/{Models,GuardCoordinator,Proofreader}.swift`, `Tests/GuardCoreTests/GuardCoordinatorTests.swift`。

**Interfaces:** `DraftSnapshot` 包含 `target: TargetIdentity`, `revision: UUID`, `text: String`；`Issue` 包含 UTF-16 `range: NSRange`, `category`, `explanation`, `replacements`。`Proofreader.check(_ snapshot: DraftSnapshot) async throws -> [Issue]`。`@MainActor GuardCoordinator` 暴露 `request(_ snapshot: DraftSnapshot)`, `invalidate()`, `authorize(current: DraftSnapshot) -> SendPermit?`, `consume(_ permit: SendPermit, current: DraftSnapshot) -> Bool`；令牌只可消费一次，检查结果匹配当前请求，完整比较身份、版本与原文。

- [ ] 写失败测试 `testRepeatedRequestDoesNotQueueSend`, `testOldResultCannotAuthorize`, `testChangedConversationCancels`, `testChangedDraftInvalidates`, `testPermitConsumedOnce`, `testFailureRequiresExplicitDecision`, `testReusedWindowDoesNotMatch`, `testPanelFocusDoesNotRetarget`。使用受控异步假引擎模拟乱序回调；断言无效状态下 `authorize == nil` 和 `consume == false`。
- [ ] 在 Mac 执行 `swift test --filter GuardCoordinatorTests`，确认测试因接口尚未实现失败，记录真实退出状态。
- [ ] 实现状态机和协议，超时、取消和引擎错误不能转换为“无问题”；面板焦点变化不更换原目标，但最终动作前必须复核真实目标。
- [ ] 重跑测试通过；增加 `testEmojiRangeValidation`，确认非法或切断组合字符的范围被拒绝且原文保留。
- [ ] 提交该逻辑单元。无 Swift 的环境仅可准备源码，不宣称此任务验收通过。

### Task 3: 接入辅助功能、蒙版和发送快捷键

**Files:** Create `Sources/GuardMac/{WeChatAdapter,PermissionMonitor,SendOverlay,KeyboardInterceptor}.swift`, `Tests/GuardMacTests/InputRoutingTests.swift`。

**Interfaces:** `@MainActor WeChatAdapter.capture() throws -> DraftSnapshot`、`replace(expected: DraftSnapshot, text: String) throws -> DraftSnapshot`、`send(expected: DraftSnapshot, permit: SendPermit) throws -> SendAttemptResult`；结果区分触发与未知，不声称送达。`PermissionMonitor` 输出 `ready/unavailable(reason)`；`SendOverlay.update(target: TargetIdentity, screenRect: CGRect?)`；`KeyboardInterceptor` 回调 `onSendIntent`，只对明确匹配的编辑框生效。

- [ ] 写路由失败测试 `testOtherAppKeyPassesThrough`, `testMarkedTextReturnPassesThrough`, `testRepeatDoesNotSendTwice`, `testPermissionRevocationInvalidates`, `testTapDisabledInvalidates`, `testOnlyMatchingPermitBypassesHook`。
- [ ] 执行 `swift test --filter InputRoutingTests`，确认失败后实现权限状态与可注入的事件判断逻辑。组合输入状态无法确定时不得声称快捷键已受保护。
- [ ] 按任务 1 的实际 AX 树实现适配器，不硬编码未知属性。使用前台窗口坐标转换构建透明 NSPanel，移位／隐藏时撤销旧覆盖；事件 tap 被停用时先标记不可用，再尝试恢复。
- [ ] 将一次发送授权在发送适配器中消费；注入事件通过来源标记与当前令牌识别，不使用全局“放行 N 毫秒”。若只能模拟鼠标，额外验证覆盖撤销期间不会放过重复物理点击，否则不支持该发送路径。
- [ ] 重跑单测，在 Mac 测试双屏、DPI、Spaces、最小化、前后台、权限撤销和中文组词。测试按钮点击和已配置快捷键均只产生一个校对请求，不自动发送。
- [ ] 提交适配单元及真实兼容性记录；不支持的宿主版本保持不可用。

### Task 4: 实现菜单栏界面与校对确认

**Files:** Create `Sources/MessageHelper/{MessageHelperApp,StatusController,ReviewPanel}.swift`, `Tests/GuardCoreTests/ReplacementTests.swift`, `Sources/GuardCore/Replacement.swift`。

**Interfaces:** `Replacement.apply(text: String, range: NSRange, replacement: String) throws -> String`；`ReviewPanel` 接收不可变快照与问题，通过 `onAccept/onSendAnyway/onCancel` 回调协调器，不能直接发送宿主消息。

- [ ] 写替换失败测试 `testEmojiNotSplit`, `testOutOfBoundsRejected`, `testCJKAndMultilinePreserved`, `testStaleSnapshotCannotReplace`。运行 `swift test --filter ReplacementTests` 确认先失败。
- [ ] 实现 UTF-16 范围验证与替换；不修改问题范围之外的文本。AX 写入前后复核身份和正文，失败时不继续发送。
- [ ] 实现“未保护／已保护／检查中／需处理／保护不可用”与暂停开关。提示界面保留“采纳并重新检查／仍然发送／取消”；错误或超时不自动放行，不记录正文。
- [ ] 重跑测试，手工验证弹出面板、切换会话、采纳后继续输入、取消和网络错误，不得把面板夺取焦点误当成新发送目标。
- [ ] 校对提供商尚未选择：只允许明确标注的测试引擎用于演示，不允许返回空问题冒充可用校对。接入真实云服务前需确定提供商、后端凭据存储、费用和文本处理政策，并由用户选择启用；该部分形成独立计划。未接入时默认不启用生产保护。
- [ ] 提交界面与替换逻辑，注明当前是保护框架还是可用校对产品。

### Task 5: 打包开发程序并执行真实验收

**Files:** Create `scripts/build-macos-app.sh`, `Resources/Info.plist`, `docs/validation/manual-checklist.md`; Modify `README.md`。

**Interfaces:** `scripts/build-macos-app.sh` 仅在 Darwin 执行，`swift build -c release` 后构建本地 `.app`；其他平台明确失败。保持稳定 bundle ID，解释开发构建权限可能随签名变化；正式分发签名和公证不冒充完成。

- [ ] 在 Mac 执行 `swift test` 与 `swift build -c release`，确认真实测试数量和退出码，不跳过失败用例。
- [ ] 执行打包脚本，启动 `.app`，完成无权限、拒绝、授权和撤销权限流程；确认无服务商配置时明确不可用。
- [ ] 使用任务 1 的受控测试会话，逐项验证蒙版、快捷键、中文候选、改稿、切换会话、重复点击、失焦、延迟回调、暂停和退出；只有用户授权的测试发送才触发真实消息。
- [ ] 验证发送状态不确定时不会自动重试；检查日志及错误输出无聊天文本。关闭助手后说明保护随之消失，不承诺不可绕过。
- [ ] README 记录安装启动命令、权限路径、支持的系统和微信版本、已知限制、真实校对接入状态，以及尚未完成的签名／公证。
- [ ] 提交验收和打包文件。未完成 Mac 真机、真实校对接入或兼容性检查时，交付明确标为原型，不称完整产品。

## 计划自检与交付边界

鼠标／快捷键双入口在任务 3；快照失效和单次授权在任务 2；纠错交互在任务 4；权限和窗口变更在任务 3、5；中文、双语及语义修正质量依赖尚未选定的真实校对服务，不由本阶段假引擎证明。市场和竞品实时核验仍未完成，不以本计划替代。

当前外部前提是可用 Mac、目标微信版本及用户主动授予的系统权限。开发者证书仅阻塞正式分发，不阻塞本地探测。后续真实模型服务接入需要独立的服务商决策和成本／隐私设计；不要求用户在聊天中提供密钥。
