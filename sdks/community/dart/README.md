# ag-ui-dart

**Agent-User Interaction (AG-UI) 协议**的 Dart SDK。

`ag-ui-dart` 为 Dart 开发者提供强类型客户端实现，用于连接兼容 AG-UI 的 Agent 服务器。基于现代 Dart 模式构建，支持健壮的校验、响应式编程和流畅的服务器发送事件流。

## 安装

```bash
dart pub add ag_ui
```

或添加到 `pubspec.yaml`：

```yaml
dependencies:
  ag_ui: ^0.3.0
```

## 特性

- 🎯 **Dart 原生** – 符合 Dart 习惯的 API，完整类型安全和空安全
- 🔗 **HTTP 连接** – `AgUiClient` 支持直连服务器和 SSE 流式传输
- 📡 **事件流** – 事件类型与官方 Python 和 TypeScript SDK 对齐（文本消息、工具调用、状态、活动、推理、生命周期等），实现实时 Agent 通信
- 🔄 **状态管理** – 自动跟踪消息/状态，支持 JSON Patch
- 🛠️ **工具交互** – 完整支持工具调用和生成式 UI
- ⚡ **高性能** – 高效事件解码，支持背压处理

## 快速示例

```dart
import 'package:ag_ui/ag_ui.dart';

// Initialize client
final client = AgUiClient(
  config: AgUiClientConfig(
    baseUrl: 'https://api.example.com',
    defaultHeaders: {'Authorization': 'Bearer token'},
  ),
);

// Create and send message
final input = SimpleRunAgentInput(
  messages: [
    UserMessage(
      id: 'msg_123',
      content: 'Hello from Dart!',
    ),
  ],
);

// Stream response events
await for (final event in client.runAgent('agentic_chat', input)) {
  if (event is TextMessageContentEvent) {
    print('Assistant: ${event.delta}');
  }
}
```

## 包结构

- **`ag_ui`** – AG-UI 协议核心客户端库
- **`ag_ui.client`** – 支持 SSE 流式传输的 HTTP 客户端
- **`ag_ui.events`** – 事件类型与事件处理
- **`ag_ui.types`** – 消息类型、工具和数据模型
- **`ag_ui.encoder`** – 事件编码/解码工具

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/dart`](https://docs.ag-ui.com/sdk/dart/client/overview)

## 核心用法

### 初始化客户端

```dart
import 'package:ag_ui/ag_ui.dart';

final client = AgUiClient(
  config: AgUiClientConfig(
    baseUrl: 'https://api.example.com',
    defaultHeaders: {'Authorization': 'Bearer token'},
    requestTimeout: Duration(seconds: 30),
  ),
);
```

### 流式接收 Agent 响应

```dart
final input = SimpleRunAgentInput(
  messages: [
    UserMessage(
      id: 'msg_${DateTime.now().millisecondsSinceEpoch}',
      content: 'Explain quantum computing',
    ),
  ],
);

await for (final event in client.runAgent('agentic_chat', input)) {
  switch (event.eventType) {
    case EventType.textMessageContent:
      final text = (event as TextMessageContentEvent).delta;
      print(text); // Stream tokens
      break;
    case EventType.runFinished:
      print('Complete');
      break;
  }
}
```

### 活动与推理事件

```dart
import 'dart:io'; // for `stderr` in the example below

await for (final event in client.runAgent('agentic_chat', input)) {
  if (event is ActivitySnapshotEvent) {
    // `content` is `Object?` — the Python reference server may emit a
    // primitive or `null`. Guard before treating it as a structured record.
    final content = event.content;
    if (content is Map<String, dynamic>) {
      // `event.replace == true`  → discard prior content for this messageId.
      // `event.replace == false` → merge/extend on top of existing content.
      print(
        'Activity (${event.activityType}, replace=${event.replace}): $content',
      );
    } else {
      // Wire-protocol surprise: log and skip rather than crash.
      stderr.writeln(
        'ActivitySnapshotEvent.content is ${content.runtimeType}, '
        'expected Map<String, dynamic>',
      );
    }
  } else if (event is ActivityDeltaEvent) {
    print('Activity patch (${event.activityType}): ${event.patch}');
  } else if (event is ReasoningMessageContentEvent) {
    print('Reasoning: ${event.delta}');
  } else if (event is ReasoningEncryptedValueEvent) {
    // Opaque cipher payload — pass through to the next agent rather than
    // attempting to decode locally.
  }
}
```

### 多模态输入

`UserMessage` 接受纯文本或有序的类型化分段列表（文本、图片、音频、视频、文档）。使用 `UserMessage.multimodal` 传入分段：

```dart
// A base64-encoded payload for an inline data part.
const base64Pdf = 'JVBERi0xLjQKJ...';

final input = SimpleRunAgentInput(
  messages: [
    UserMessage.multimodal(
      id: 'msg_${DateTime.now().millisecondsSinceEpoch}',
      parts: [
        TextInputContent('What is in this image?'),
        ImageInputContent(
          // UrlSource.mimeType is optional; DataSource requires it.
          source: UrlSource(
            value: 'https://example.com/photo.png',
            mimeType: 'image/png',
          ),
        ),
        DocumentInputContent(
          source: DataSource(value: base64Pdf, mimeType: 'application/pdf'),
        ),
      ],
    ),
  ],
);
```

`content` getter 对纯文本消息返回文本，对多模态消息返回 `null`；读取 `messageContent` 获取类型化联合体。

默认的 `UserMessage({content})` 构造函数不是 `const`，因为它在运行时将字符串包装为 `TextContent`。使用 `UserMessage.fromContent` 保持编译期常量——如果你之前使用 `const UserMessage(content: '...')`，这也是迁移路径：

```dart
// Before (no longer const):
// UserMessage(id: 'u-1', content: 'Hello')

// After — const-friendly:
const msg = UserMessage.fromContent(
  id: 'u-1',
  messageContent: TextContent('Hello'),
);
```

### 基于工具的交互

```dart
List<ToolCall> toolCalls = [];

// Collect tool calls from first run
await for (final event in client.runToolBasedGenerativeUi(input)) {
  if (event is MessagesSnapshotEvent) {
    for (final msg in event.messages) {
      if (msg is AssistantMessage && msg.toolCalls != null) {
        toolCalls.addAll(msg.toolCalls!);
      }
    }
  }
}

// Process tool calls and send results
final toolResults = toolCalls.map((call) => ToolMessage(
  id: 'tool_${DateTime.now().millisecondsSinceEpoch}',
  toolCallId: call.id,
  content: processToolCall(call),
)).toList();

final followUp = SimpleRunAgentInput(
  threadId: input.threadId,
  messages: [...input.messages, ...toolResults],
);

// Get final response
await for (final event in client.runToolBasedGenerativeUi(followUp)) {
  // Handle response
}
```

### 状态管理

```dart
Map<String, dynamic> state = {};
List<Message> messages = [];

await for (final event in client.runSharedState(input)) {
  switch (event.eventType) {
    case EventType.stateSnapshot:
      state = (event as StateSnapshotEvent).snapshot;
      break;
    case EventType.stateDelta:
      // Apply JSON Patch (RFC 6902) operations
      applyJsonPatch(state, (event as StateDeltaEvent).delta);
      break;
    case EventType.messagesSnapshot:
      messages = (event as MessagesSnapshotEvent).messages;
      break;
  }
}
```

### 错误处理

Dart SDK 的错误构成 [`AGUIError`](https://pub.dev/documentation/ag_ui/latest/ag_ui/AGUIError-class.html) 下的单一层级。捕获该基类可统一处理所有错误；捕获下方特定子类可针对性恢复。通过 [`EventDecoder`](https://pub.dev/documentation/ag_ui/latest/ag_ui/EventDecoder-class.html)，线路解码侧抛出 [`DecodingError`]；客户端请求/传输层抛出 [`TransportError`] 和 [`ValidationError`]；取消表现为 [`CancellationError`]。

```dart
final cancelToken = CancelToken();

try {
  await for (final event in client.runAgent('agent', input, cancelToken: cancelToken)) {
    // Process events
    if (shouldCancel(event)) {
      cancelToken.cancel();
      break;
    }
  }
} on TransportError catch (e) {
  print('Connection error: ${e.message}');
} on DecodingError catch (e) {
  print('Decode error: ${e.message}');
} on ValidationError catch (e) {
  print('Validation error: ${e.message}');
} on CancellationError {
  print('Request cancelled');
} on AGUIError catch (e) {
  // Catch-all for any AG-UI-originated error (covers
  // AGUIValidationError thrown directly from a `Type.fromJson` call
  // when the event isn't routed through the EventDecoder pipeline).
  print('AG-UI error: $e');
}
```

> **取消说明：** `CancelToken.cancel()` 停止向你的流投递事件，但**不会**中止底层 HTTP socket。连接在服务器关闭或操作系统空闲超时触发时释放。如需真正中止连接，请为每个请求提供自定义 `IOClient`。

### 代理说明：线路拼写规范化

Dart SDK 在每个 `fromJson` 工厂上同时接受 **camelCase**（TypeScript 规范，如 `threadId`、`runId`、`parentRunId`、`encryptedValue`、`rawEvent`）和 **snake_case**（Python 规范，如 `thread_id`、`run_id`、`parent_run_id`、`encrypted_value`、`raw_event`），但 `toJson` 始终输出 **camelCase**——没有选择输出 snake_case 的选项。

如果你将 Dart SDK 用作代理，介于输出 snake_case 的 Python 服务器和仅接受 snake_case 的严格消费者之间，必须在边界处将键转换回去。TypeScript 和 Python 官方 SDK 都容忍输入中的 camelCase 形式，因此实践中很少有问题——但严格的 snake_case 消费者在技术上符合协议，会从 Dart 中间层看到规范化后的载荷。

在单次 `BaseEvent.rawEvent` 往返中，拼写由读取两个键（`rawEvent` / `raw_event`）的辅助方法保留；Dart 侧的 camelCase 输出是唯一的规范化点。

## 完整示例

```dart
import 'dart:io';
import 'package:ag_ui/ag_ui.dart';

void main() async {
  // Initialize client from environment
  final client = AgUiClient(
    config: AgUiClientConfig(
      baseUrl: Platform.environment['AGUI_BASE_URL'] ?? 'http://localhost:8000',
      defaultHeaders: Platform.environment['AGUI_API_KEY'] != null
          ? {'Authorization': 'Bearer ${Platform.environment['AGUI_API_KEY']}'}
          : null,
    ),
  );

  // Interactive chat loop
  stdout.write('You: ');
  final userInput = stdin.readLineSync() ?? '';

  final input = SimpleRunAgentInput(
    messages: [
      UserMessage(
        id: 'msg_${DateTime.now().millisecondsSinceEpoch}',
        content: userInput,
      ),
    ],
  );

  stdout.write('Assistant: ');
  await for (final event in client.runAgent('agentic_chat', input)) {
    if (event is TextMessageContentEvent) {
      stdout.write(event.delta);
    } else if (event is ToolCallStartEvent) {
      print('\nCalling tool: ${event.toolCallName}');
    } else if (event.eventType == EventType.runFinished) {
      print('\nDone!');
      break;
    }
  }

  client.dispose();
}
```

## 从 0.1.0 迁移

0.2.0 对直接构造事件的调用者引入一个源码破坏性变更：

- **`ToolCallResultEvent.role` 现在为 `ToolCallResultRole?` 而非 `String?`。** 更新直接构造：

  ```dart
  // Before (0.1.0)
  ToolCallResultEvent(
    messageId: '...',
    toolCallId: '...',
    content: '...',
    role: 'tool',
  );

  // After (0.2.0)
  ToolCallResultEvent(
    messageId: '...',
    toolCallId: '...',
    content: '...',
    role: ToolCallResultRole.tool,
  );
  ```

  线路解码不受影响：线路上未知的 `role` 字符串通过 `ToolCallResultRole.fromString` 吸收，并回退到 `ToolCallResultRole.tool` 以保持前向兼容。完整理由见 [`CHANGELOG.md`](CHANGELOG.md) 的 "Breaking Changes"。

- **`TimeoutError` 已重命名为 `AGUITimeoutError`**，以避免遮蔽 `dart:async.TimeoutError`（由 `Future.timeout(...)` / `Stream.timeout(...)` 抛出）。裸名称保留为已弃用的 typedef 别名，将在 1.0.0 中移除：

  ```dart
  // Before (0.1.0)
  } on TimeoutError catch (e) { /* ... */ }

  // After (0.2.0)
  } on AGUITimeoutError catch (e) { /* ... */ }
  ```

  如果同时导入 `package:ag_ui/ag_ui.dart` 和 `dart:async`，请使用新名称以避免符号冲突，并确保原始 `dart:async.TimeoutError` 实例（任何 `.timeout(...)` 调用中非常常见）不会被针对 SDK 类型的 `on TimeoutError` 分支静默吸收。

  反向情况说明：如果你之前本意是 `dart:async.TimeoutError`，却因 `package:ag_ui/ag_ui.dart` 的 `TimeoutError` 在非限定名称解析中胜出而意外捕获了 SDK 实例，重命名会暴露此前的冲突。迁移到 `AGUITimeoutError` 后，裸 `TimeoutError` 分支现在明确指向 `dart:async.TimeoutError`——运行时行为随之改变。

`THINKING_TEXT_MESSAGE_*` 事件类型在 0.2.0 中也已弃用，改用规范的 `REASONING_*` 事件；解码支持保留至 1.0.0。迁移映射见 `CHANGELOG.md` 的 "Deprecated"。

## 错误

SDK 暴露一个小型错误层级，按来源有意拆分：

- `AGUIError` — SDK 全局根。捕获 `on AGUIError` 覆盖 SDK 能抛出的所有错误：运行时、传输、解码以及直接工厂校验。需要单一兜底时使用。
- `AgUiError` — 继承 `AGUIError`。覆盖运行时/传输/解码：`TransportError`、`AGUITimeoutError`、`CancellationError`、`DecodingError` 以及客户端 `ValidationError`。需要限定为"SDK 遇到运行时问题"但明确不想捕获直接工厂校验错误时使用。（`TimeoutError` 保留为 `AGUITimeoutError` 的已弃用别名；优先使用新名称以避免遮蔽 `dart:async.TimeoutError`。）
- `AGUIValidationError` — 继承 `AGUIError`（非 `AgUiError`）。由线路解码边界的 `*.fromJson` 工厂构造函数抛出。事件流经 `EventDecoder` 时会被包装为 `DecodingError`，因此使用解码器流水线的消费者不会直接看到它。直接调用工厂（`TextMessageStartEvent.fromJson(...)`）的调用者会看到。
- `EncoderError` 及其子类型（`DecodeError`、`EncodeError`、编码器侧 `ValidationError`）继承 `AGUIError`。`EventDecoder` 流水线原样重新抛出这些错误，以便调用者按类型模式匹配。

使用 `EventDecoder` 的生产代码推荐捕获配方：

```dart
try {
  for (final event in stream) { handle(event); }
} on DecodingError catch (e) {
  // Wire-format problem — log e.field, e.expectedType, e.actualValue.
} on TransportError catch (e) {
  // HTTP / SSE transport failure.
} on AgUiError catch (e) {
  // Anything else from the runtime/transport family.
} on AGUIError catch (e) {
  // Catch-all (would also catch direct-factory AGUIValidationError if you
  // ever bypass the decoder).
}
```

## 示例

参见 [`example/`](example/) 目录：
- 用于测试 AG-UI 服务器的交互式 CLI
- 基于工具的生成式 UI 流程
- 消息流式模式
- 完整端到端演示

## 测试

```bash
# Run unit tests
dart test

# Run integration tests (requires server)
cd test/integration
./helpers/start_server.sh
dart test
./helpers/stop_server.sh
```

## 贡献

欢迎贡献！请：
1. Fork 仓库
2. 创建功能分支
3. 为新功能添加测试
4. 确保所有测试通过
5. 提交 Pull Request

## 密文数据保留

部分 AG-UI 事件（`ReasoningEncryptedValueEvent`、`ReasoningMessage`、`ToolMessage`）携带不透明密文载荷，必须在 Agent 之间逐字转发。本 SDK 围绕这些载荷实现纵深防御：

**成功路径** — 每个 `BaseEvent` 上的 `raw