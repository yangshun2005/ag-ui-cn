---
name: agui-dotnet-human-in-the-loop
description: >
  使用 AG-UI .NET SDK 暂停 AG-UI 智能体运行以交由人工处理，然后恢复运行——将敏感工具置于显式审批之后，或中断运行以在继续前收集用户的自由输入。适用场景：在工具执行前要求人工审批（ApprovalRequiredAIFunction + ToolApprovalRequestContent/ToolApprovalResponseContent，通过追加请求 + CreateResponse(approved) 恢复）；暂停运行以向用户提问，并用其回答恢复（InterruptRequestContent / InterruptResponseContent、RunAgentInput.Resume）；暂停的运行如何以 RUN_FINISHED outcome=interrupt 呈现，以及客户端如何检测并应答。不适用场景：在服务端无需审批即可运行的工具（使用 agui-dotnet-server-tools）或在客户端运行的工具（使用 agui-dotnet-client-tools）；普通聊天（使用 agui-dotnet-streaming-chat）；共享状态、生成式 UI、多模态或 protobuf。
---

# AG-UI .NET — 人在回路（审批与中断）

目标：在决策点停止智能体运行，将控制权交给人工，并用其决策恢复同一运行——要么批准敏感工具调用，要么提供智能体所请求的输入。

暂停的运行以 `RUN_FINISHED { outcome: interrupt }` 结束当前轮次。客户端检查响应，获取人工决策，然后发送携带答案的后续轮次；SDK 将其重新连接到暂停的工具调用，使运行继续。

## 审批：将工具置于人工是/否之后

这是简单路径，无需自定义端点代码——`ToChatRequestContext` 处理恢复。

### 服务端：包装敏感工具

将函数包装在 `ApprovalRequiredAIFunction` 中。当模型调用它时，函数调用客户端会抛出 `ToolApprovalRequestContent` 而不执行：

```csharp
using Microsoft.Extensions.AI;

var deleteFile = new ApprovalRequiredAIFunction(
    AIFunctionFactory.Create(DeleteFile, "delete_file", "Deletes a file."));

builder.Services.AddChatClient(/* provider IChatClient */)
    .ConfigureOptions(o => (o.Tools ??= []).Add(deleteFile))
    .UseFunctionInvocation();
```

### 客户端：检测请求、决策、恢复

第一轮返回 `ToolApprovalRequestContent`。将其展示给人工，然后通过追加请求和人工响应并再次流式传输来恢复：

```csharp
using Microsoft.Extensions.AI;

var messages = new List<ChatMessage> { new(ChatRole.User, "Delete report-draft.txt") };

var turn1 = new List<ChatResponseUpdate>();
await foreach (var u in client.GetStreamingResponseAsync(messages))
{
    turn1.Add(u);
}

var request = turn1.SelectMany(u => u.Contents)
                   .OfType<ToolApprovalRequestContent>()
                   .FirstOrDefault();

if (request is { ToolCall: FunctionCallContent call })
{
    bool approved = AskHuman($"Run {call.Name}?");   // your UI / prompt

    messages.Add(new ChatMessage(ChatRole.Assistant, [request]));
    messages.Add(new ChatMessage(ChatRole.User, [request.CreateResponse(approved)]));

    await foreach (var u in client.GetStreamingResponseAsync(messages))
    {
        Console.Write(u.Text);   // runs the tool if approved, skips it if not
    }
}
```

在恢复的轮次中，SDK 会重新配对审批请求和响应，使函数调用客户端执行（或跳过）底层调用。

## 中断：在运行中途向用户请求输入

当智能体需要自由输入（而非是/否）时，用 `InterruptRequestContent` 暂停，并用 `InterruptResponseContent` 恢复。客户端侧与审批类似：

```csharp
using AGUI.Abstractions;

var interrupt = turn1.SelectMany(u => u.Contents)
                     .OfType<InterruptRequestContent>()
                     .FirstOrDefault();

if (interrupt is not null)
{
    var answer = AskHuman(interrupt.Message);   // e.g. a username
    var payload = JsonSerializer.SerializeToElement(new { response = answer });

    messages.Add(new ChatMessage(ChatRole.Assistant, [interrupt]));
    messages.Add(new ChatMessage(ChatRole.User,
        [new InterruptResponseContent(interrupt.RequestId) { Payload = payload }]));

    await foreach (var u in client.GetStreamingResponseAsync(messages))
    {
        Console.Write(u.Text);
    }
}
```

`AGUIChatClient` 在传输时将响应编码为 `RunAgentInput.Resume[]`。中断请求携带 `Reason`（例如 `InterruptReasons.InputRequired`）、用于显示的 `Message`，以及描述预期载荷的可选 `ResponseSchema`。

在服务端，通过从 `DelegatingChatClient` 产出携带内容的更新来抛出中断。通常你会将模型工具调用桥接为中断：检测模型的 `FunctionCallContent`，改为发出 `InterruptRequestContent(call.CallId)`，并在恢复时将 `InterruptResponseContent` 重写回模型期望的 `FunctionCallContent` + `FunctionResultContent` 对，使其像工具返回了用户答案一样继续。

## 反模式

- **仅用响应内容恢复。** 请求（`ToolApprovalRequestContent` / `InterruptRequestContent`）和匹配的响应都必须出现在恢复的消息列表中，且 id 相同；SDK 将它们配对以重新连接到暂停的调用。仅发送响应会使运行没有可恢复的内容。
- **在提示词中指示模型请求确认。** 审批门是结构性的——被包装的工具会自行暂停运行。同时告诉模型"请用户确认"的提示词会产生冗余的文本问题和第二次往返。应告诉模型直接调用工具，让审批门处理审批。

## 验证

1. 第一轮结束时不执行副作用：流以 `RUN_FINISHED { outcome: interrupt }` 结束，且响应包含 `ToolApprovalRequestContent`（或 `InterruptRequestContent`）。
2. 以批准恢复后，工具的效果发生且运行正常结束；以拒绝恢复后，效果不发生且模型在没有该效果的情况下继续。
3. 对于输入中断，智能体的最终答案反映人工提供的值。