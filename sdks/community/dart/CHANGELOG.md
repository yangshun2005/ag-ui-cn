# 更新日志

本文件记录 AG-UI Dart SDK 的所有重要变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)，
本项目遵循 [语义化版本](https://semver.org/spec/v2.0.0.html)。

## [0.3.0]

### 破坏性变更（评审修复轮）
- **`StateDeltaEvent.delta` 和 `ActivityDeltaEvent.patch` 现在为
  `List<Map<String, dynamic>>`，不再是 `List<dynamic>`。** RFC 6902 JSON
  Patch 操作始终是对象。使用 `requireListField<Map<String,
  dynamic>>` 会在解码器边界将非对象元素暴露为 `AGUIValidationError`，
  并附带 `field: 'delta[$i]'` / `field: 'patch[$i]'` 索引，
  而不是在首次访问 `op['op']` 时泄漏下游的 `TypeError`。
  已直接使用 `event.delta[i]` 并已转换为 Map 的消费者不受影响；
  将列表存储为 `List<dynamic>` 的消费者需要更新类型注解。
  **迁移：** 将 `event.delta` / `event.patch` 上的 `List<dynamic>` 类型注解
  改为 `List<Map<String, dynamic>>`。已直接访问 `op['op']` / `op['path']`
  且未显式转换的代码无需改动。
- **`SseParser.maxDataBytes` 重命名为 `maxDataCodeUnits`。** 该字段
  实际度量的是 UTF-16 码元而非字节——重命名纠正了误导性名称。
  `SseParser(maxDataBytes: ...)` 调用点必须更新为
  `SseParser(maxDataCodeUnits: ...)`。

### 修复（评审修复轮）
- **`ActivityMessage.fromJson` 现在静默剥离 `encryptedValue` /
  `encrypted_value`，不再抛出 `AGUIValidationError`。** `ActivityMessage`
  在规范协议中不是 `BaseMessage` 的扩展，因此该字段不适用。Dart 是唯一
  在遇到该字段时中断流的 SDK；TS 静默剥离（zod 默认行为），Python 保留。
  此变更恢复了代理发出该字段时的前向兼容性。
- **`ReasoningEncryptedValueEvent.fromJson` 不再将密文载荷存入
  `BaseEvent.rawEvent`。** 此前 `rawEvent: _readRawEvent(json)`
  将完整线上 JSON（含 `encryptedValue`）存入继承的 `rawEvent` 字段，
  使所有错误路径中的密文清理失效。该事件类型的 `rawEvent` 现在始终为
  `null`；需要原始线上形式的代理应在调用 `fromJson` 前保留。
- **`RunStartedEvent.fromJson` 重新抛出时不再将出错的载荷附加到
  `AGUIValidationError.json`。** 完整外层载荷（以及内层
  `RunAgentInput`，它可通过 `input.messages[*]` 携带 `encryptedValue`）
  均被省略，因此密文数据无法通过验证错误泄漏——与
  `MessagesSnapshotEvent.fromJson` 中已有的清理一致。
- **`MessagesSnapshotEvent.fromJson` 重新抛出时现在完全丢弃 `json:`。**
  此前转发 `e.json` 会在外层错误上暴露内层 Message map；对于 Tool/Reasoning
  子类型，该 map 可能携带 `encryptedValue`。丢弃 `json:` 以匹配
  `AssistantMessage.fromJson` 的工具调用 IIFE，后者已使用谨慎的默认值。
- **`JsonDecoder.requireEitherField` 现在区分"键存在但为 null"与
  "键缺失"。** 此前两种情况产生相同的
  "Missing required field 'X' (or 'Y')" 消息，误导消费者以为在 camelCase
  键显式为 null 时 snake_case 别名可能生效。现在：键存在但为 null 产生
  "Required field 'X' is present but null"；两者均缺失仍产生双键错误。
- **`copyWith` 哨兵值清理完成。** `ThinkingStartEvent.title`、
  `ToolCallResultEvent.role`、`StateSnapshotEvent.snapshot` 和
  `RunErrorEvent.code` 现在使用 `kUnsetSentinel` 模式，调用方可通过
  `copyWith(field: null)` 清除这些可空字段。载荷字段的"已知对等差距"
  列表现已为空。
- **`EventEncoder.acceptsProtobuf` 和 `EventDecoder.decodeBinary` 现在带有
  明确的 dartdoc 警告**，说明 protobuf 尚未端到端实现。协商
  `application/vnd.ag-ui.event+proto` 的客户端会收到误导性的
  "Invalid UTF-8 data" 错误；文档现在引导消费者在 protobuf 支持落地前
  使用 SSE 传输。
- **`groupRelatedEvents` dartdoc 现在记录 `ReasoningStart` /
  `ReasoningEnd` 的不对称性。** 阶段级推理事件作为独立单例发出；
  只有消息级 `REASONING_MESSAGE_*` 事件被分组。需要将阶段级标记与消息组
  关联的消费者必须自行跟踪阶段边界。
- **`processChunk` 在 for 循环后重置 `errorRoutedInChunk`。** 该标志
  此前仅在循环内设置；循环体之后的未来抛出点可能静默吞掉无关错误。
- **`SseParser` 错误消息已更正。** OOM 防护错误现在说
  "code-unit limit"（而非 "byte limit"），以匹配上限实际度量的内容。
- **`SseParser._processField` 现在对 `data:` 间分隔符使用 `write('\\n')`
  而非 `writeln()`。** `writeln()` 在所有 Dart 平台上等价，但显式形式
  消除了是否发出平台行终止符的任何歧义。
- **`EventType.fromString` dartdoc 已加强**，附明确契约说明：调用方不得
  更改抛出类型 `ArgumentError`，因为 `BaseEvent.fromJson` 使用窄化的
  `on ArgumentError` 捕获来区分未知事件类型与工厂缺陷。

### 修复（评审轮——协议对等）
- **`encryptedValue` 现在贯通所有 BaseMessage 子类型**
  （`DeveloperMessage`、`SystemMessage`、`AssistantMessage`、
  `UserMessage`），位于基类 `Message` 上。对齐规范 TS
  `BaseMessageSchema.encryptedValue: z.string().optional()` 和 Python
  `BaseMessage.encrypted_value: Optional[str]`。此前该字段
  仅存在于 `ToolMessage` 和 `ReasoningMessage`，因此 Dart 代理解码
  来自 TS 或 Python 服务器、其 assistant 或 user 消息携带
  `encryptedValue` 的 `MESSAGES_SNAPSHOT` 时，会在解码时静默丢弃
  该值，无法在下一跳重新发出。解码接受 `encryptedValue`（TS 规范）
  和 `encrypted_value`（Python 规范）；`toJson` 发出 camelCase；
  各子类型的 `copyWith` 通过哨兵模式接受显式 null 清除。
  `ToolMessage` 和 `ReasoningMessage` 的字段声明已移除，改为从基类
  继承——线上形状不变。
- **`raw_event`（snake_case）现在在每个事件工厂中保留。**
  所有约 30 个 `BaseEvent` 子类现在通过集中式 `_readRawEvent` 辅助方法
  读取 `rawEvent`，使用 `containsKey` 优先级：camelCase 键存在时胜出
  （即使显式为 `null`），仅当 camelCase 缺失时才查询 snake_case 键。
  此前每个工厂直接读取 `json['rawEvent']`，静默丢弃 Python 风格的
  `raw_event` 载荷。`toJson` 仍仅发出 camelCase。
- **`REASONING_ENCRYPTED_VALUE` 不再拒绝空 `entityId` / `encryptedValue`
  字符串。** 规范 TS 使用 `z.string()`，Python 使用 `str`——两者均不
  施加最小长度。Dart 独有的空字符串拒绝（在
  `ReasoningEncryptedValueEvent.fromJson` 和 `EventDecoder.validate` 中）
  过于严格，会拒绝规范 SDK 接受的载荷。严格子类型判别器保留——
  未知子类型仍抛出。
- **`SseParser._processField` 现在对空前导 `data:` 行和重复 `event:` 行
  匹配 WHATWG SSE 规范。** `data:` 分支使用 `_dataBuffer.isNotEmpty`
  作为"是否已写入数据？"的启发式判断，将 `data:\ndata: x\n\n` 折叠为
  `"x"` 而非规范正确的 `"\nx"`。现在使用 `_hasDataField` 标志
  （镜像 `EventStreamAdapter.appendDataLine` 中的 `inDataBlock` 模式）。
  `event:` 分支此前在每行 `event:` 上追加；按规范必须替换。
- **`EventStreamAdapter.fromRawSseStream` 现在将下游取消、暂停和恢复
  传播到上游原始 SSE 订阅。** 此前上游 `rawStream.listen(...)`
  订阅是即发即忘——提前取消适配流的消费者会使上游无限期排空
  （长生命周期 agent 流上的真实资源泄漏）。
- **`SseParser.parseBytes` 现在在流关闭时刷新任何最终未终止事件。**
  通过 `parseLines` 路由，使 `parseLines` 中最终的 `_dispatchEvent()`
  刷新对字节流源也触发。此前未以尾随空行结束的字节源会丢失最后一条
  缓冲消息。
- **`copyWith` 哨兵值清理。** `RawEvent.source`、
  `RunAgentInput.state`、`RunAgentInput.forwardedProps` 和
  `Run.result` 此前使用标准 `?? this.field` 模式，调用方无法通过
  `copyWith(field: null)` 清除它们。现在使用现有哨兵模式。下方的
  "已知对等差距"列表已更新。
- **`JsonDecoder.optionalEitherField` 现在基于键存在性解析，
  而非值非 null。** 同时携带 `camelKey: null` 和
  `snake_key: <value>` 的载荷此前会落到 snake_case 值；
  `requireEitherField` 的文档契约是 camelCase 键存在时胜出（即使显式
  为 `null`）。实现现在与 dartdoc 一致。`context.dart` 中内联的
  `forwardedProps` 解码已迁移到相同的 `containsKey` 规则以保持一致。
- **`ToolMessage.fromJson` 和 `ToolResult.fromJson` 现在使用
  `requireEitherField`**，替代旧的
  `optionalEitherField + 手动 null 检查 + 自定义抛出`模式，
  与已为 `RunAgentInput.fromJson` 和 `Run.fromJson` 完成的迁移一致。
- **`Validators.validateMessageContent` 现在仅接受 `String`。**
  0.2.0 之前的宽松 `Map`/`List` 分支是死代码（SDK 中无调用方传入这些
  类型），且与规范 `BaseMessage.content: Optional[str]` 不一致。
  多模态 `UserMessage.content` 仍是跟踪中的对等差距。
- **`Validators.validateUrl` 现在拒绝包含 C0 控制字符或 DEL 的 URL**
  （`\x00`–`\x1f`、`\x7f`）。`Uri.parse` 对嵌入的 `\n` / `\r` / `\t`
  过于宽松，可能作为头注入向量流入 HTTP 请求行。
- **`JsonDecoder.requireField` 和 `optionalField` 的转换失败路径现在
  在将内部异常包装为 `AGUIValidationError` 时保留 `cause: e`。**
  此前结构化 cause 仅通过 `'$e'` 插值扁平化到消息中。

### 文档
- `AGUIValidationError.json` dartdoc 现在带有明确的敏感数据警告：
  该字段捕获整个线上载荷，包括密文字段。`toString()` 不发出它
  （默认安全），但某些日志框架使用的基于反射的序列化器会泄漏。
  在发送到外部 sink 的日志行上优先使用 `.field` 和 `.value`。
- `EventDecoder.validate` dartdoc 现在记录双源错误类不对称性：
  `validate()` 抛出 `client/errors.dart` 的 `ValidationError`；
  `fromJson` 侧的急切拒绝抛出 `types/base.dart` 的
  `AGUIValidationError`。两者通过公共 `decode` / `decodeJson` 边界
  统一表现为 `DecodingError`；两者均扩展 `AGUIError`。
- `BaseEvent.rawEvent` dartdoc 现在注明往返发出后果——赋给此字段的
  任何内容将在下次 `encode` 时重新发出。如果代理不希望上游载荷回显到
  下游，请在飞行事件上设置 `rawEvent: null`。
- README 新增"代理说明：线上拼写规范化"段落，记录 SDK 在 `fromJson`
  上接受 camelCase 和 snake_case，但在 `toJson` 上始终发出 camelCase。
  错误处理部分已刷新，使用当前错误层级类名（`TransportError`、
  `DecodingError`、`ValidationError`、`CancellationError`，
  均在 `AGUIError` 下）。
- `AgUiClient.runAgent` dartdoc 的 `Throws:` 列表已刷新以匹配当前
  错误层级。
- `EventStreamAdapter.groupRelatedEvents` dartdoc 现在带有明确的
  无界状态警告——开放组（已收到 `*Start` 但 `*End` 尚未到达）会保留在
  内存中，直到匹配的结束事件或流完成。同样的注意事项适用于
  `accumulateTextMessages`。

### 修复（评审轮——行为）
- **`Tool.copyWith(parameters: null)` 现在正确清除 `parameters`。**
  此前 `parameters ?? this.parameters` 模式在传入 `null` 时静默保留
  现有值；该字段现在使用 `_unsetTool` 哨兵模式，与
  `ToolCall.encryptedValue` 及 SDK 中所有其他可空字段一致。此差距
  在 0.2.0 "已知对等差距"列表中遗漏——已在此更正。
- **`EventStreamAdapter.fromRawSseStream` 现在惰性订阅上游**
  （在 `controller.onListen` 内），而非在调用时急切订阅。
  此前获取返回流但从未订阅的调用方会泄漏上游 SSE 连接，直到服务器
  关闭。上一评审轮添加的取消、暂停和恢复传播保留；订阅生命周期回调
  现在使用空安全 `?.` 调用，因为订阅不再是 `late final`。
- **`Message.fromJson` 现在在 `MessageRole.fromString` 失败时
  在 `AGUIValidationError` 中保留线上 JSON 载荷。** 此前抛出错误时
  未设置 `json:`，无法识别 `MESSAGES_SNAPSHOT` 中哪条消息的角色
  无法识别。重新抛出的错误携带原始 `json` map，使解码器流水线能以
  完整上下文将其暴露为 `DecodingError`。

### 变更
- `TimeoutError` 重命名为 `AGUITimeoutError`，避免遮蔽内置的
  `dart:async.TimeoutError`（由 `Future.timeout(...)` /
  `Stream.timeout(...)` 抛出）。裸名称保留为已弃用的 typedef 别名
  以向后兼容，将在 1.0.0 移除。`AgUiClient` 中的内部调用点直接抛出
  新名称。README "Errors" 配方和"从 0.1.0 迁移"部分已说明重命名，
  以便同时使用 `package:ag_ui/ag_ui.dart` 和 `dart:async` 的消费者
  避免符号冲突。
- `TEXT_MESSAGE_CONTENT`、`TOOL_CALL_ARGS` 和
  `REASONING_MESSAGE_CONTENT` 现在接受空 `delta`，`TOOL_CALL_RESULT`
  接受空 `content`，以匹配规范 TS/Python schema（`z.string()` / `str`
  无 `min(1)` 约束）。此前 Dart SDK 在 `fromJson` 工厂和
  `EventDecoder.validate` 流水线两处均拒绝空值；合法发出故意空块
  （例如 noop 内容刷新）的 Python 或 TS 服务器在 Dart 中解码失败，
  但在规范 SDK 中通过。`REASONING_ENCRYPTED_VALUE` 上的空密文载荷
  （`entityId`、`encryptedValue`）继续被拒绝——"密文载荷无优雅默认值"
  契约保留。

### 修复
- `ToolCall` 现在携带可选 `encryptedValue` 字段，以对齐规范 TS
  （`ToolCallSchema.encryptedValue: z.string().optional()`）
  和 Python（`ToolCall.encrypted_value: Optional[str]`）。此前到达的
  消息带有 `toolCalls: [{..., encryptedValue: "..."}]` 时会在解码时
  静默丢弃该值，无法在代理跳转时重新发出。解码接受 `encryptedValue`
  和 `encrypted_value`；`toJson` 在存在时发出 camelCase 键；
  `copyWith` 使用哨兵模式，调用方可通过
  `copyWith(encryptedValue: null)` 显式清除。
- `RunAgentInput` 现在携带可选 `parentRunId` 字段，以对齐规范 TS
  （`RunAgentInputSchema.parentRunId: z.string().optional()`）
  和 Python（`RunAgentInput.parent_run_id`）。此前带有
  `input.parentRunId: '...'` 的 `RUN_STARTED` 载荷解码时该字段被
  静默丢弃，尽管 `RunStartedEvent.parentRunId` 本身被