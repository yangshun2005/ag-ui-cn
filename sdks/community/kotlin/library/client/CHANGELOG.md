# 更新日志

## 0.5.0 — 2026-11-05

- 客户端已对齐 AG-UI 1.0，包括运行输入处理和状态应用。
- 发送给 agent 的运行输入中不再包含活动消息。
- 新增 REASONING_* 事件支持；THINKING_* 已弃用，并提供 ReplaceWith 提示。
- 新增 wasmJs（浏览器）和 macosArm64 目标。
- 状态补丁以非递归方式应用，避免深层嵌套状态导致栈溢出。
- 在各历史路径中保留多模态工具内容、元数据和子 agent 归属信息。
- 已记录：格式错误的 STATE_DELTA 会导致流程致命失败，而无法应用的 delta 通过 onStateError 报告。

### 破坏性变更

- StateManager 现在会拒绝此前接受的 delta：替换/移除不存在的成员、在不存在的父级下添加、移动到自身的子级，以及不带值添加。
- 被拒绝的 delta 报告 `com.agui.client.state.JsonPatchApplicationException`，而非 kotlin-json-patch 的异常类型。
- MAX_JSON_DEPTH 从 512 降至 128；嵌套过深的 SSE 载荷会被拒绝。
- 从 commonMain 中移除了 `catch (e: Error)`；嵌套过深的 SSE 载荷会被拒绝，而非被吞掉。
- 活动消息不再在运行输入中回传。
- 数值 delta 比较现在按值进行（1 等于 1.0）。
- Group ID 改为 com.ag-ui.community；minSdk 降至 24；Kotlin 固定为 2.1.20。