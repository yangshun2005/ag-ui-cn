# 更新日志

## 0.1.1 — 2026-09-09

- 在 `Role` 枚举中新增 `REASONING("reasoning")`，使 `Role.fromValue("reasoning")` 能够成功调用。
- 在密封的 `Message` 层级中新增 `ReasoningMessage` 类型，用于在对话历史中表示推理内容。
- `ReasoningMessage` 携带可选的 `encryptedValue`，可在多次往返中保留加密的思维链推理内容。
- 保留现有的 `(id, content)` 和 `(id, content, name)` 构造函数；`ReasoningMessageStartEvent` 现在可以携带 `role: "reasoning"` 字段。

### 破坏性变更

- 密封的 `Message` 层级新增 `ReasoningMessage` 变体；对 `Message` 的穷尽式 switch 必须处理该变体。