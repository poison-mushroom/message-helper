# Message Helper

macOS 消息发送前校对助手：通过发送快捷键拦截与发送按钮蒙版，在用户确认后放行一次发送。首个适配目标暂定桌面微信，后续探索飞书、钉钉。

当前处于文档设计阶段，尚无可运行 App；跨应用读取、拦截与发送能力需在 Mac 上验证，不承诺全应用覆盖。

## 开发文档

- [需求背景与产品需求](docs/requirements.md)：目标场景、范围、功能与验收要求。
- [macOS 实现方案](docs/implementation.md)：本地准备、能力探测、模块接口、权限及开发顺序。
- [逐任务开发计划](docs/superpowers/plans/2026-10-21-macos-send-guard.md)：保护框架的详细实现步骤。
- [前期市场与竞品分析](docs/research/2026-10-21-product-discovery.md)：初步假设与待核实资料，非已完成的实时市场调查。

先阅读前两份文档，再在 Mac 上验证微信的 Accessibility 能力。开发、测试和打包命令是实现后的目标，目前尚未提供对应源码和脚本。
