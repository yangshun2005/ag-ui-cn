# AG-UI .NET SDK 架构

## 本 SDK 的用途

AG-UI（Agent User Interface，智能体用户界面）是一种协议，用于将事件从 AI 智能体后端流式传输到前端 UI。.NET SDK 提供两项能力：**服务端托管层**，可将任意 `IChatClient` 变成 AG-UI 端点；**客户端库**，可将这些端点作为标准 `IChatClient` 消费。两端共享同一套协议类型。

核心设计理念是：**你不必学习新的编程模型。** 如果你已用 `Microsoft.Extensions.AI` 与语言模型交互，只需把同一个 `IChatClient` 接入 AG-UI 托管层，SDK 负责转换。协议是传输机制，不是应用抽象。

## 包结构

```
            AGUI.Abstractions       (协议类型)
            ↑      ↑       ↑
   AGUI.Formatting  |   AGUI.Server
   (线格式)          |   (服务端适配器)
        ↑           |
   AGUI.Protobuf    |
   (protobuf 编解码) |
        ↑   ↑       |
        AGUI.Client (消费端)
```

**`AGUI.Abstractions`** 定义协议：事件、消息、工具、能力及其 JSON 序列化（`AGUIJsonSerializerContext`）。`Generated/` 下的模型类由 `spec/1.0/schema.json` 生成（`pnpm --filter @ag-ui/spec generate`）——请改 schema，不要改它们；转换器、`AGUIContent` 联合类型及序列化上下文仍为手写。它不依赖 HTTP、ASP.NET Core 或任何托管框架。其他所有包都依赖它。它还承载 `AGUIJsonUtilities.RegisterInterruptContentTypes`，即仅用于 STJ 的中断内容类型注册。

**`AGUI.Formatting`** 定义线格式抽象：`IAGUIEventStreamFormatter`（双向读写格式化器）和 `SseEventStreamFormatter`（Server-Sent Events 线格式）。它仅依赖 Abstractions 和 `System.Net.ServerSentEvents`。

**`AGUI.Protobuf`** 提供 protobuf 编解码器（`internal` 的 `AGUIProtobuf`）、公开的 `ProtobufEventStreamFormatter`，以及 `JsonElement`↔`google.protobuf.Value` 桥接。生成的 proto 类型为 `internal`；`.proto` schema 引用自 `sdks/typescript/packages/proto`（不复制），确保线格式与 `@ag-ui/proto` 同步。它依赖 Abstractions、Formatting 和 `Google.Protobuf`。

**`AGUI.Server`** 是**框架无关**的服务端适配器（原 `AGUI.Hosting.AspNetCore` 更名而来，已剥离全部 ASP.NET）。它提供扩展方法，将 `ChatResponseUpdate` 流（任意 `IChatClient` 的输出）转换为 AG-UI 事件流：`ToChatRequestContext`/`ChatRequestContext`、`AsAGUIEventStreamAsync`、`AGUIStreamOptions` 和 `AGUIConstants`。它仅引用 Abstractions 和 `Microsoft.Extensions.AI.Abstractions`——**不含 ASP.NET**。

**`AGUI.Client`** 是消费 AG-UI 服务端时使用的包。它将 HTTP 连接包装为 `IChatClient` 实现（`AGUIChatClient`，由 `AGUIChatClientOptions` 构造），调用方无需知道自己在与 AG-UI 端点通信。它还包含传输层（`AGUIHttpTransport`/`IAGUITransport`）和传输协商原语（`AGUIEventStreamHandler`，一个公开的 `DelegatingHandler`，以及 `ReadAGUIEventStreamAsync`），调用方可将其接入自己的 `HttpClient` 以请求 protobuf。它依赖 Formatting（并间接依赖 Abstractions）。

服务端与客户端**刻意保持独立。** 服务端包从不引用客户端包；二者仅共享协议类型和格式化器。**`src/` 中没有任何项目引用 `Microsoft.AspNetCore.App`**——ASP.NET 托管胶水（`AGUIResults`、`AGUIEventStreamResult`、`MapAGUI` 端点扩展和 `AddAGUI` DI 注册）仅存在于 `samples/AGUI.Samples.Shared` 项目，这是唯一带有 `Microsoft.AspNetCore.App` 的 `FrameworkReference` 的项目。

| 包 | 依赖 | 目标框架 |
|---------|-------------|-------------------|
| AGUI.Abstractions | `System.Text.Json`、`M.E.AI.Abstractions` | net10.0、net9.0、net8.0、netstandard2.0、net472 |
| AGUI.Formatting | AGUI.Abstractions、`System.Net.ServerSentEvents` | net10.0、net9.0、net8.0、netstandard2.0、net472 |
| AGUI.Protobuf | AGUI.Abstractions、AGUI.Formatting、`Google.Protobuf` | net10.0、net9.0、net8.0、netstandard2.0、net472 |
| AGUI.Client | AGUI.Abstractions、AGUI.Formatting、`M.E.AI`、`System.Net.ServerSentEvents` | net10.0、net9.0、net8.0、netstandard2.0、net472 |
| AGUI.Server | AGUI.Abstractions、`M.E.AI.Abstractions` | net10.0、net9.0、net8.0 |
| *samples/AGUI.Samples.Shared* | AGUI.Server、AGUI.Protobuf、`Microsoft.AspNetCore.App` | net10.0 |

所有 `src/` 包均兼容 AOT。每个可序列化类型都注册在 `AGUIJsonSerializerContext`（源生成的 `JsonSerializerContext`）中，任何序列化路径都不依赖运行时反射。

---

## 服务端端点的工作方式

最简单的 AG-UI 服务端就是一个 POST 端点。参见入门示例：

1. 端点从请求体接收 `RunAgentInput`，包含线程 ID、运行 ID、对话消息、客户端提供的工具定义及可选状态。
2. 调用 `input.ToChatRequestContext(jsonSerializerOptions, streamOptions?)`。返回的 `ChatRequestContext` 携带适配后的 `ChatMessage` 列表（`ctx.Messages`）、配置好的 `ChatOptions`（`ctx.ChatOptions`，原始输入存于 `AdditionalProperties[AGUIConstants.RunAgentInputKey]`，客户端声明的工具已走审批流管道）以及流转换器选项。
3. 调用 `chatClient.GetStreamingResponseAsync(ctx.Messages, ctx.ChatOptions, ct)`，得到 `IAsyncEnumerable<ChatResponseUpdate>`。
4. 通过 `.AsAGUIEventStreamAsync(ctx, ct)` 将其转换为 `IAsyncEnumerable<BaseEvent>`。
5. 将事件流交给协商结果。示例中为 `AGUIResults.Events(events, httpContext, ct)`（来自 `samples/AGUI.Samples.Shared`，经 `app.MapAGUI("/")` 映射），它检查请求的 `Accept` 头，将流写为 Server-Sent Events（默认）或 protobuf。

就这么简单。SDK 处理事件生命周期（发出 `RunStartedEvent` 和 `RunFinishedEvent`）、跟踪文本消息开关状态、映射工具调用与结果，并将一切序列化为线格式。混合服务端/客户端工具调用管道（将客户端工具包装为 `ApprovalRequiredAIFunction`、在续接轮次注入合成的 `ToolApprovalResponseContent`、在线格式上解包）内置于 `ToChatRequestContext` 和 `AsAGUIEventStreamAsync`——调用方无需显式配置。

示例在此基础上逐步构建：

- **Step 01（入门）** 展示上述最小端点。
- **Step 02（后端工具）** 添加服务端工具：定义为普通 C# 方法，用 `AIFunctionFactory.Create` 包装，与 `ToChatRequestContext` 安装的客户端工具一起加入 `ctx.ChatOptions.Tools`。`IChatClient` 管道（配合 `UseFunctionInvocation()`）自动调用它们。SDK 将工具调用/结果事件流式发送到前端展示。
- **Step 03（前端工具）** 展示相反方向：*客户端*定义工具并随请求发送。服务端将其传给 LLM，SDK 将工具调用事件流回前端执行。续接时客户端工具结果通过 `RunAgentInput.Messages` 带回，SDK 注入匹配的审批响应，使内部 `FunctionInvokingChatClient` 正确恢复。
- **Step 05（状态管理）** 展示如何用 `StateSnapshotEvent` 和 `StateDeltaEvent` 向前端推送结构化状态。
- **Step 07（思考事件）** 展示托管层原生将 `TextReasoningContent`（MEAI 内容类型）转换为 AG-UI 推理事件链（`ReasoningStartEvent` → `ReasoningMessageStartEvent` → `ReasoningMessageContentEvent` → `ReasoningMessageEndEvent` → `ReasoningEndEvent`）。无需配置 `MapContent`；示例只是一个产生推理增量的 `IChatClient`。
- **Step 09（中断 – 审批）** 展示内置工具审批机制：当 `IChatClient` 管道为真正需要审批的服务端工具产生 `ToolApprovalRequestContent` 时，SDK 自动发出 outcome 为 `"interrupt"` 的 `RunFinishedEvent`。客户端用 `AGUIResume` 恢复，服务端从断点继续。
- **Step 13（Protobuf）** 启用 protobuf 传输：服务端注册 `ProtobufEventStreamFormatter`，客户端将 `AGUIEventStreamHandler` 接入 `HttpClient` 以声明 protobuf 媒体类型，同一协商端点便以长度前缀 protobuf 而非 SSE 编码事件。
- **Step 14（遥测）** 通过同一管道接入 OpenTelemetry 追踪。两个进程都在 `IChatClient` 上调用 `UseOpenTelemetry()`，并订阅 `Experimental.AGUI.Server`/`Experimental.AGUI.Client` 和 `Experimental.Microsoft.Extensions.AI` 源。服务端将每个产生的事件流包装在 `agui.run` span 中（GenAI `chat` span 看不到的 AG-UI 协议层）；客户端在同一对话活动下驱动与其他步骤相同的三个工具场景，使客户端 HTTP span 与服务端 run span 共享同一 W3C trace。默认导出到控制台；设置 `OTEL_EXPORTER_OTLP_ENDPOINT` 时导出到 OTLP（如 Aspire dashboard 或 OpenTelemetry Collector）。

### 传输与内容协商

事件流独立于线格式。格式化器（`AGUI.Formatting` 中的 `IAGUIEventStreamFormatter`）读写一种具体表示：

- **Server-Sent Events**（`SseEventStreamFormatter`，`text/event-stream`）是默认传输，始终可用。
- **Protobuf**（`ProtobufEventStreamFormatter`，`application/vnd.ag-ui.event+proto`，来自 `AGUI.Protobuf`）需显式启用。服务端通过将格式化器注册为 `IAGUIEventStreamFormatter` 启用；客户端通过 `Accept` 头声明。

服务端上，协商的 `AGUIResults.Events` 结果根据请求 `Accept` 头选择格式化器（回退到 SSE）。客户端上，`AGUIChatClient` 声明接受的格式，并解码服务端返回的格式。protobuf schema 直接引用自 `sdks/typescript/packages/proto`，两种传输都与 `@ag-ui/proto` 线兼容。由于两种格式都解码为相同的 `BaseEvent`/`ChatResponseUpdate` 流，应用代码和测试与传输无关。

### 使用 AGUIStreamOptions 进行工具映射

`AGUIStreamOptions` 是传给 `ToChatRequestContext` 并由 `AsAGUIEventStreamAsync` 消费的配置对象。它仅提供方法——无公开 getter/setter；一切通过流式配置。

最常见用例是状态管理。假设智能体有 `write_document` 工具写 markdown 文档，你希望前端在写入时实时预览。没有工具映射，你得手动拦截流并注入 `StateSnapshotEvent`。使用工具映射：

```csharp
var streamOptions = new AGUIStreamOptions()
    .MapResultAsStateSnapshot("write_document");

var ctx = input.ToChatRequestContext(jsonSerializerOptions, streamOptions);
```

这告诉 SDK：当该工具的 `ToolCallResultEvent` 到达时，同时发出以工具结果为快照载荷的 `StateSnapshotEvent`。

更高级场景中，`MapCall` 和 `MapResult` 接受任意回调：

- **`MapCall(toolName, mapper)`** 在工具调用事件（start/args/end）发出后触发。回调接收 `FunctionCallContent` 并返回额外事件。Dojo `predictive_state_updates` 示例用它逐 token 流式发送部分状态快照，让前端在工具执行完成前就能实时预览。
- **`MapResult(toolName, mapper)`** 在 `ToolCallResultEvent` 后触发。回调接收 `FunctionResultContent`。

两个便捷方法覆盖常见情况：

- **`MapResultAsStateSnapshot(toolName)`** 发出 `StateSnapshotEvent`。
- **`MapResultAsStateDelta(toolName)`** 发出 `StateDeltaEvent`。

Dojo 服务端的 `CreateAgenticUIStreamOptions` 展示了二者：`create_plan` 将结果映射为状态快照（完整计划），`update_plan_step` 映射为状态增量（单步的增量更新）。

### 自定义内容与中断

`AGUIStreamOptions` 上两个流式方法为转换器不原生处理的内容类型安装回退链：

- **`MapInterrupt(Func<AIContent, AGUIInterrupt?>)`** 注册回调，接收 `AIContent` 并可返回 `AGUIInterrupt`。若返回非 null，SDK 发出 outcome 为 `"interrupt"` 的 `RunFinishedEvent` 并停止运行。客户端应向用户展示中断、收集响应，并通过 `RunAgentInput.Resume` 恢复。多次注册按顺序链式执行，首个非 null 结果生效。
- **`MapContent(Func<AIContent, IEnumerable<BaseEvent>?>)`** 注册回调，接收 `AIContent` 并可返回 `BaseEvent` 序列。这是产生自有内容类型的智能体框架的扩展点——例如将 `TextReasoningContent` 映射为推理事件，或将工作流步骤标记映射为 `StepStartedEvent` / `StepFinishedEvent`。多次注册同样链式执行。

---

## 事件协议

每次 AG-UI 交互形状相同：客户端 POST `RunAgentInput`，服务端以 SSE 流回事件序列。事件类型均定义在 `AGUI.Abstractions` 中：

**运行生命周期。** 每个流以 `RunStartedEvent` 开始，以 `RunFinishedEvent` 结束（失败时为 `RunErrorEvent`）。`RunFinishedEvent` 携带可选 outcome 字符串——智能体需要用户输入时为 `"interrupt"`，调用了前端工具时为 `"tool_calls"`，正常完成为 null。

**文本消息。** 智能体文本输出以 `TextMessageStartEvent`（分配消息 ID 和角色）、一个或多个 `TextMessageContentEvent`（各携带文本增量）和 `TextMessageEndEvent` 流式发送。这种括号结构让客户端增量渲染文本并知道消息何时完成。

**工具调用。** 工具调用以 `ToolCallStartEvent`（工具名和调用 ID）、`ToolCallArgsEvent`（序列化参数）、`ToolCallEndEvent` 流式发送，可选 `ToolCallResultEvent`（若工具在服务端执行）。前端工具没有结果事件——客户端执行工具并在下一个 `RunAgentInput` 中回传结果。

**状态。** `StateSnapshotEvent` 用新 `JsonElement` 值替换完整前端状态。`StateDeltaEvent` 传递增量补丁。状态 schema 由智能体定义——协议只负责携带。

**活动。** `ActivitySnapshotEvent` 和 `ActivityDeltaEvent` 与状态事件类似，但用于瞬态 UI 指示（进度条、加载动画、状态文本）。

**推理。** 模型暴露思维链输出时，服务端用 `ReasoningStartEvent` / `ReasoningEndEvent` 包裹，内部用 `ReasoningMessageStartEvent` / `ReasoningMessageContentEvent` / `ReasoningMessageEndEvent` 承载实际文本。`ReasoningEncryptedValueEvent` 携带客户端可回传的不透明推理 token。

**步骤。** `StepStartedEvent` 和 `StepFinishedEvent` 标记多步智能体中的命名阶段边界。

**消息快照。** `MessagesSnapshotEvent` 用服务端视图替换客户端对话历史。

**扩展性。** `CustomEvent` 携带自由格式的 name 和 value。`RawEvent` 原样包装 `JsonElement`——SDK 内部用它让智能体框架将预构建协议事件注入 `ChatResponseUpdate` 流。

**中断。** `AGUIInterrupt` 是附加到 outcome 为 `"interrupt"` 的 `RunFinishedEvent` 的载荷，携带 ID、原因字符串和任意 `JsonElement` 载荷。最常见的中断是工具审批：服务端填入 `AGUIToolApprovalPayload`（包装 `AGUIToolCallInfo`），让 UI 向用户展示智能体想做什么。客户端用包含中断 ID 和响应载荷的 `AGUIResume` 响应。

---

## 消息与请求载荷

`RunAgentInput` 是客户端启动运行时发送的 JSON 体，包含：

- `ThreadId` 和 `RunId` — 客户端生成的标识符。
- `Messages` — 对话历史，为 `AGUIMessage` 对象。
- `Tools` — 客户端提供的工具定义，为 `AGUITool` 对象。
- `State` — 可选 `JsonElement`，携带当前前端状态。缺失表示"无状态"，按契约裸 JSON `null` 含义相同——.NET 将二者读入未设置的 `JsonElement?`，重新序列化时省略该键，这是所有 SDK 一致的行为（见 `sdks/fixtures/null-omission.json`）。
- `Resume` —