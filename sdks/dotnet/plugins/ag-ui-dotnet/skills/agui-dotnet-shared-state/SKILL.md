---
name: agui-dotnet-shared-state
description: >
  使用 AG-UI .NET SDK 在 AG-UI 智能体与其客户端之间共享结构化、持续演进的状态——客户端在请求中提供初始状态，服务端读取、修改该状态，并在聊天过程中以快照或增量形式将更新后的状态流式返回。适用场景：通过 ChatOptions.RawRepresentationFactory -> RunAgentInput.State 从客户端发送初始/工作状态；在服务端使用 ChatOptions.TryGetRunAgentInput 和 RunAgentInput.State 读取入站状态；在 DelegatingChatClient 中通过 ChatResponseUpdate.RawRepresentation 发出完整状态对象（StateSnapshotEvent）或增量 JSON-Patch 变更（StateDeltaEvent）；在客户端通过 update.RawRepresentation 以 StateSnapshotEvent 形式读取状态。不适用场景：传递一次性工具参数（请使用 agui-dotnet-server-tools）；从工具调用渲染 UI 组件（请使用 agui-dotnet-generative-ui）；普通聊天（请使用 agui-dotnet-streaming-chat）；审批/中断、多模态或 protobuf。
---

# AG-UI .NET — 共享状态

目标：在客户端与智能体之间保持一个结构化对象（表单、文档、计划、食谱）同步——客户端提供当前状态，智能体更新它，新状态与助手文本一起流式返回。

状态在传输中位于 `RunAgentInput.State`（入站）以及 `StateSnapshotEvent` / `StateDeltaEvent`（出站）内。在 `Microsoft.Extensions.AI` 的语境下，入站状态搭载在请求的原始表示上，出站状态搭载在 `ChatResponseUpdate` 的原始表示上。

## 客户端：提供初始状态并读取回传

随请求发送起始状态，并监听响应中的快照：

```csharp
using System.Text.Json;
using AGUI.Abstractions;
using Microsoft.Extensions.AI;

var initialState = JsonSerializer.SerializeToElement(new
{
    recipe = new { title = "", ingredients = Array.Empty<string>(), steps = Array.Empty<string>() }
});

var options = new ChatOptions
{
    RawRepresentationFactory = _ => new RunAgentInput { State = initialState },
};

var messages = new List<ChatMessage> { new(ChatRole.User, "Suggest an Italian pasta recipe") };

JsonElement? latestState = null;
await foreach (var update in client.GetStreamingResponseAsync(messages, options))
{
    if (update.RawRepresentation is StateSnapshotEvent snapshot)
    {
        latestState = snapshot.Snapshot;   // the full updated state object
    }
    Console.Write(update.Text);            // the assistant's summary streams as usual
}
```

状态更新是**无内容**的——它们通过 `update.RawRepresentation` 到达，而非 `update.Text`。

## 服务端：读取入站状态并发出新状态

在流水线中添加一个 `DelegatingChatClient`，读取 `RunAgentInput.State`、生成新状态，并以 `StateSnapshotEvent` 形式产出：

```csharp
using System.Runtime.CompilerServices;
using System.Text.Json;
using AGUI.Abstractions;
using AGUI.Server;
using Microsoft.Extensions.AI;

internal sealed class RecipeStateChatClient(IChatClient inner, JsonSerializerOptions jso)
    : DelegatingChatClient(inner)
{
    public override async IAsyncEnumerable<ChatResponseUpdate> GetStreamingResponseAsync(
        IEnumerable<ChatMessage> messages,
        ChatOptions? options = null,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        if (options?.TryGetRunAgentInput(out var input) is true
            && input!.State is { ValueKind: JsonValueKind.Object } incoming)
        {
            var newState = await BuildStateAsync(messages, incoming, ct);   // your logic / an LLM call

            yield return new ChatResponseUpdate
            {
                RawRepresentation = new StateSnapshotEvent
                {
                    Snapshot = JsonSerializer.SerializeToElement(newState, jso.GetTypeInfo(typeof(AgentState))),
                },
            };
        }

        await foreach (var update in base.GetStreamingResponseAsync(messages, options, ct))
        {
            yield return update;   // the assistant's text summary
        }
    }
}
```

在函数调用之后注册它：

```csharp
builder.Services.AddChatClient(/* provider IChatClient */)
    .UseFunctionInvocation()
    .Use((inner, sp) => new RecipeStateChatClient(
        inner, sp.GetRequiredService<IOptions<JsonOptions>>().Value.SerializerOptions));
```

`TryGetRunAgentInput` 从端点构建的 `ChatOptions` 中恢复原始的 `RunAgentInput`——服务端聊天客户端正是借此访问 `State`（以及线程 id、工具等），无需端点额外传递。

## 快照与增量

- **`StateSnapshotEvent`** 携带完整状态对象。简单且可自我校正；当整个状态发生变化或需要重新同步时发送。
- **`StateDeltaEvent`** 携带针对最后已知状态的 JSON Patch（RFC 6902）变更数组。适用于对大型对象进行小而频繁的编辑，避免重发全部内容。客户端将补丁应用到自己的副本上。

## AOT 安全的状态类型

状态对象通过源生成的上下文进行序列化。将每个状态类型放入 `JsonSerializerContext`，在主机上注册（`ConfigureHttpJsonOptions(o => o.SerializerOptions.TypeInfoResolverChain.Add(...))`），并通过 `jso.GetTypeInfo(typeof(T))` 而非反射进行序列化。

## 反模式

- **聚合响应而非流式处理。** `StateSnapshotEvent` / `StateDeltaEvent` 通过无内容更新到达，非流式聚合会丢弃它们。应在流式响应的 `await foreach` 内读取状态。
- **针对客户端从未收到的状态发送增量。** `StateDeltaEvent` 会修补客户端最后已知的状态；如果客户端错过了先前的快照，补丁将无法应用。应以完整的 `StateSnapshotEvent` 开始会话（或重新同步），然后再发送增量。

## 验证

1. 客户端发送非空 `State`，运行至少流式返回一个 `StateSnapshotEvent`，其 `Snapshot` 反映智能体的更改。
2. 助手文本和状态在同一次运行中到达；读取 `update.RawRepresentation` 得到 `StateSnapshotEvent`，而 `update.Text` 得到摘要。
3. 若使用增量：将流式 JSON Patch 应用到先前状态，产生的对象应与完整快照相同。