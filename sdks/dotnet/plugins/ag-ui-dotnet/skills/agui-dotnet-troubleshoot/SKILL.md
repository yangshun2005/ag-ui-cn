---
name: agui-dotnet-troubleshoot
description: >
  诊断并修复基于 AG-UI SDK 构建的 AG-UI .NET 应用中的问题——当流式聊天、工具、状态、中断或传输出现异常时使用。适用场景：智能体忘记之前的轮次 / 历史被截断；update.ConversationId 始终为 null；状态或生命周期更新从未到达；工具在运行时抛出异常或产生 AOT/裁剪警告；运行中途出现未知工具错误；端点返回 406 Not Acceptable；中断或审批无法恢复；以及检查原始 AG-UI 事件流（curl SSE 帧、ChatResponseUpdate.RawRepresentation 上的 RawEvent）以查看实际传输内容。不适用场景：首次设置或正常流程（请使用 agui-dotnet-streaming-chat）；干净地实现某个特定功能（请使用对应的 agui-dotnet-* 技能）。
---

# AG-UI .NET — 故障排查

目标：将症状映射到原因，并在 AG-UI .NET 应用中修复。

## 智能体忘记之前的轮次 / 历史被截断

服务器是无状态的，轮次之间不保留任何内容，因此客户端必须在每轮重新发送完整历史。如果只发送最新消息，模型会丢失所有先前上下文。

修复：维护一个持续累积的 `List<ChatMessage>`，追加助手回复（`history.AddMessages(updates.ToChatResponse())`），并重新发送整个列表。若要让服务器持有对话，请在各轮之间复用同一个 `ChatOptions` 实例以保持线程 id 稳定。

## `update.ConversationId` 始终为 null

这是设计使然——SDK 从不暴露服务端对话 id。非 null 的 `ConversationId` 会导致 `Microsoft.Extensions.AI` 包装器只发送增量，从而在无状态服务器上截断历史。

修复：不要通过 `ConversationId` 跟踪对话。使用 `RUN_STARTED` 事件中的线程 id（`update.RawRepresentation as RunStartedEvent`），或直接复用同一个 `ChatOptions` 以保持一个稳定的线程。

## 状态、推理或生命周期更新从未到达

这些更新依附于**无内容**的更新——`update.RawRepresentation`（如 `StateSnapshotEvent`、`RunStartedEvent`、`RawEvent` 等）且不含文本。非流式的 `GetResponseAsync` 聚合会丢弃不携带消息内容的更新。

修复：使用流式 API（对 `GetStreamingResponseAsync` 使用 `await foreach`），在循环内检查 `update.RawRepresentation` / `update.Contents`，而不是聚合为单个 `ChatResponse`。

## 工具在运行时抛出异常或产生 AOT/裁剪警告

参数或结果类型为非基元的工具需要源生成的序列化；基于反射的 Schema/绑定在 Native AOT 和裁剪下会失败。

修复：将工具的参数和结果类型放入 `JsonSerializerContext`，在主机上注册（`ConfigureHttpJsonOptions(o => o.SerializerOptions.TypeInfoResolverChain.Add(...))`），并传给 `AIFunctionFactory.Create(method, serializerOptions: ctx.Options)`。

## 运行中途出现未知工具调用错误

当客户端声明了一个服务器不拥有的工具时，服务器的函数调用客户端会尝试执行它并失败——除非被告知该调用属于别处。

修复：为服务器聊天客户端注册 `.UseFunctionInvocation(fic => fic.TerminateOnUnknownCalls = true)`。未知的（客户端）工具随后会干净地结束服务器轮次，并转发给客户端执行。

## 端点返回 406 Not Acceptable

传输是内容协商的。406 表示请求的 `Accept` 头要求了服务器无法生成的格式（例如未注册 protobuf 格式化器却请求 protobuf，或 `Accept` 中排除了 `text/event-stream`）。

修复：让客户端接受服务器提供的格式。对于 SSE，允许 `text/event-stream`（或 `*/*`）；对于 protobuf，在服务器上注册 `ProtobufEventStreamFormatter`，并在客户端的 `AGUIEventStreamHandler` 中列出它。

## 中断或审批无法恢复

恢复暂停的运行需要**同时**在恢复的消息列表中包含请求和匹配的响应，且共享相同的 id——SDK 会将它们配对以重新连接到暂停的调用。

修复：同时追加两者——助手消息上的 `ToolApprovalRequestContent`（或 `InterruptRequestContent`）及其在用户消息上的 `CreateResponse(...)` / `InterruptResponseContent(requestId)`——然后再次流式传输。仅发送响应会导致没有可恢复的内容。

## 检查实际传输内容

使用 curl 直接查看原始事件帧：

```bash
curl -N -X POST http://localhost:5000/ \
  -H "Content-Type: application/json" -H "Accept: text/event-stream" \
  -d '{"threadId":"t1","runId":"r1","messages":[{"id":"m1","role":"user","content":"Hi"}]}'
```

健康的运行会显示 `RUN_STARTED` … `TEXT_MESSAGE_CONTENT` … `RUN_FINISHED`。暂停的运行以 `RUN_FINISHED { outcome: interrupt }` 结束。工具运行会显示 `TOOL_CALL_START` / `TOOL_CALL_ARGS` / `TOOL_CALL_END` / `TOOL_CALL_RESULT`。

在代码中，遥测和自定义服务器事件会以 `RawEvent` 形式出现在更新上：

```csharp
if (update.RawRepresentation is RawEvent { Source: var source, Event: var payload })
{
    Console.WriteLine($"[{source}] {payload.GetRawText()}");
}
```

## 验证修复

应用修复后，重现原始提示，并对照传输流确认症状已消失——例如第二轮现在能反映第一轮（历史），`StateSnapshotEvent` 现在能到达（流式），或响应的 `Content-Type` 现在与协商的传输匹配（406）。