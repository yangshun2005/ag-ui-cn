# 更新日志

## 0.0.6 — 2026-10-13

- 委托日志现在遵循 debug 设置；仅当 `debug.enabled` 为 true 时，A2A 任务和回复才通过 `console.debug` 记录，而不再始终记录。

### 破坏性变更

无。

## 0.0.5 — 2026-10-13

- 支持按请求克隆 agent 的运行时（如 CopilotKit）：`clone()` 现在会继承 agent 卡片、客户端、指令和编排器。此前，克隆后的首次运行在读取 agent 卡片时会抛错。
- 委托的 `send_message_to_a2a_agent` 调用不再报 "Tool arguments not found" 错误：其参数从本次运行的流中读取，这也避免了编排器被重复订阅。

### 破坏性变更

无。

## 0.0.4 — 2026-10-12

- 要求 `@ag-ui/client` `>=1.0.0`；移除了未使用的 `@ag-ui/core` peer 依赖。
- 移除了未使用的 `@ag-ui/core/schemas` 导入，因此该包不再从 `@ag-ui/core` 导入任何内容。

### 破坏性变更

无。

## 0.0.3

- 在 npm 包中发布声明的 MIT 许可证及许可证文件。