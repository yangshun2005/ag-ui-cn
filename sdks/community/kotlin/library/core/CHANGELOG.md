# 更新日志

## 0.5.0 — 2026-11-05

- SDK 对齐 AG-UI 1.0，包括严格的事件/输入解码和状态补丁结构校验。
- 新增 REASONING_* 事件；THINKING_* 事件和 ThinkingTelemetryState 已弃用，并提供 ReplaceWith 提示。
- RUN_FINISHED 结果新增中断协议支持（`Interrupt`、`ResumeStatus`、`ResumeEntry`）。
- 新增 ACTIVITY_SNAPSHOT/ACTIVITY_DELTA 事件、ActivityMessage、Role.ACTIVITY、ToolMessage.error 及多模态 UserMessage 支持。
- core、client 和 tools 新增 wasmJs（浏览器）和 macosArm64 目标。
- 修复 JSON Pointer 校验，避免长指针在 JVM、Wasm 和 Native 上导致栈溢出。
- RunFinishedSuccessOutcome 现在精确序列化为 `{"type":"success"}`。
- 工具结果内容数组序列化不再回退为字符串。

### 破坏性变更

- 目标为 AG-UI 1.0；严格的事件/输入解码和收紧的校验可能拒绝此前接受的载荷。
- 使用量计数按精确的安全 JSON 整数校验；小数/指数写法会被规范化。
- 嵌套类型模型及消息/结果名称中的显式 null 检查已收紧。
- THINKING_* 事件、ThinkingTelemetryState 和 AgentState.thinking 已弃用；请迁移至 REASONING_*/ReasoningTelemetryState。
- 移除 kotlin-json-patch 依赖；StateManager 改用内部 RFC 6902 实现。
- Group ID 变更为 com.ag-ui.community；移除 com.contextable/com.agui 引用。
- minSdk 降至 24；Kotlin 固定为 2.1.20。
- RunFinishedSuccessOutcome 序列化现在仅输出 type 键；请确认消费方能容忍额外字段。