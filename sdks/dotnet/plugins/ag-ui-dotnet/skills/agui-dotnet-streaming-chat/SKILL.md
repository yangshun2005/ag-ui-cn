---
name: agui-dotnet-streaming-chat
description: >
  开始使用 AG-UI .NET SDK：使用 AG-UI .NET NuGet 包（AGUI.Client、
  AGUI.Server、AGUI.Formatting、AGUI.Abstractions）引导并运行你的第一个流式聊天
  应用（客户端 + 服务端）。适用场景：安装哪些包以及如何连接它们；针对端点构造 AGUIChatClient
  并以 Microsoft.Extensions.AI IChatClient /
  ChatResponseUpdate 流式接收回复；基于任意 IChatClient 自行托管 AG-UI POST 端点；
  运行单轮或多轮对话；无状态代理（客户端持有历史，每轮重发）与托管型 / 持有会话的代理（服务端
  保留会话，客户端固定线程 id）。不适用场景：修改 AG-UI
  .NET SDK 本身（使用 agui-dotnet-* 贡献者技能）；服务端工具
  （agui-dotnet-server-tools）；客户端工具（agui-dotnet-client-tools）；中断 /
  人在回路；共享状态；生成式 UI；多模态；或 protobuf 传输。
---

# AG-UI .NET 入门 — 流式聊天

目标：安装包、启动 AG-UI 端点、连接客户端，并流式传输多轮对话。**当你运行客户端并看到它打印出流式回复时，即告完成。**

## 两种代理类型（先读这里）

对话如何保持延续，取决于**谁持有历史**：

- **无状态代理（默认）。** 服务端在轮次之间不保留任何内容——它是请求的纯函数。**客户端**持有历史，并在每轮重发完整消息列表。从这里开始。
- **托管型 / 持有会话的代理。** 服务端（或运行时）将以**线程 id** 为键的会话持久化，从而能回忆历史。客户端在稳定的线程 id 下发送新一轮。见[下文](#托管型代理)。

`AGUIChatClient` 是一个 `Microsoft.Extensions.AI.IChatClient`，因此你可以用常规的 `GetStreamingResponseAsync(messages, options, ct)` API 来使用它。

**线程 id。** 每个对话都有一个——它是整个交互的稳定标识符。通过**复用同一个 `ChatOptions` 实例**在各轮之间保持其稳定：`AGUIChatClient` 在首轮之后会将解析出的线程 id 固定到该实例上。（SDK 从不将其暴露为 `ConversationId`——`update.ConversationId` 按设计始终为 `null`，见 issue #4869——因此不要用 `ConversationId` 来跟踪对话。）

## 安装包

客户端应用：

```bash
dotnet add package AGUI.Client
```

服务端应用：

```bash
dotnet add package AGUI.Server      # RunAgentInput 适配 + 事件流转换
dotnet add package AGUI.Formatting  # SseEventStreamFormatter（写入 SSE 线格式）
```

`dotnet add package` 安装最新的稳定版本。要查找版本或固定某个版本：

```bash
dotnet package search AGUI.Client --exact-match   # 显示 nuget.org 上的最新版本
dotnet add package AGUI.Client --prerelease        # 安装最新版，包括预发布版
dotnet add package AGUI.Client --version <x.y.z>   # 固定特定版本
```

`AGUI.Abstractions`（协议类型，如 `RunAgentInput` 和 `RunStartedEvent`）会随两者传递引入。在服务端你还会用到 `Microsoft.Extensions.AI`（`AddChatClient`、`IChatClient`）以及你选择的聊天客户端提供程序（OpenAI、Azure OpenAI、Ollama 等）。

## 无状态：客户端

从 `AGUIChatClientOptions` 构造它（这里通过 `(HttpClient, endpoint)` 选项构造函数）：

```csharp
using AGUI.Client;
using Microsoft.Extensions.AI;

using var httpClient = new HttpClient();
IChatClient client = new AGUIChatClient(new(httpClient, "http://localhost:5001"));

await foreach (var update in client.GetStreamingResponseAsync(
    new List<ChatMessage> { new(ChatRole.User, "Hello") }))
{
    Console.Write(update.Text); // 文本增量在到达时流式输出
}
```

你持有 `HttpClient`。传输默认使用 Server-Sent Events。

## 无状态：服务端

自行托管端点：接收 `RunAgentInput`，将其适配为 Microsoft.Extensions.AI，从任意 `IChatClient` 流式输出，转换为 AG-UI 事件，并以 SSE 写出。

```csharp
using AGUI.Abstractions;
using AGUI.Formatting;
using AGUI.Server;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.Options;
using JsonOptions = Microsoft.AspNetCore.Http.Json.JsonOptions;

var builder = WebApplication.CreateBuilder(args);

// 注册你的聊天客户端。任何 Microsoft.Extensions.AI 提供程序均可
// （OpenAI、Ollama、本地模型等）；此处以 Azure OpenAI 为例。
builder.Services.AddChatClient(
    new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
        .GetChatClient(deploymentName)
        .AsIChatClient());

// SseEventStreamFormatter 是无状态的——注册为单例一次即可。
builder.Services.AddSingleton<IAGUIEventStreamFormatter, SseEventStreamFormatter>();

var app = builder.Build();

app.MapPost("/", async (
    [FromBody] RunAgentInput input,
    IChatClient chatClient,
    IAGUIEventStreamFormatter formatter,
    IOptions<JsonOptions> jsonOptions,
    HttpContext http,
    CancellationToken ct) =>
{
    var jso = jsonOptions.Value.SerializerOptions;

    // 1) RunAgentInput -> messages + ChatOptions
    var ctx = input.ToChatRequestContext(jso);

    // 2) 从模型流式输出
    var updates = chatClient.GetStreamingResponseAsync(ctx.Messages, ctx.ChatOptions, ct);

    // 3) ChatResponseUpdate 流 -> AG-UI 事件流
    var events = updates.AsAGUIEventStreamAsync(ctx, ct);

    // 4) 使用注入的（单例）格式化器写出事件
    http.Response.ContentType = formatter.MediaType;
    http.Response.Headers.CacheControl = "no-cache";
    await formatter.WriteAsync(events, http.Response.Body, ct);
});

app.Run();
```

这就是服务端的全部契约。如果之后要暴露服务端执行的工具，在聊天客户端注册处添加 `.UseFunctionInvocation()`——纯聊天不需要。

## 无状态：多轮（携带历史）

无状态服务端没有记忆，因此**客户端**维护运行中的 `List<ChatMessage>`：追加助手回复，然后添加下一轮用户消息。复用同一个 `ChatOptions`，使线程 id 在整个对话中保持稳定。

```csharp
var messages = new List<ChatMessage> { new(ChatRole.User, "Hello") };
var options = new ChatOptions();

await SendAsync(messages, options);

messages.Add(new ChatMessage(ChatRole.User, "How are you?"));
await SendAsync(messages, options);

async Task SendAsync(List<ChatMessage> history, ChatOptions options)
{
    var updates = new List<ChatResponseUpdate>();
    await foreach (var update in client.GetStreamingResponseAsync(history, options))
    {
        Console.Write(update.Text);          // 在回复到达时流式输出
        updates.Add(update);
    }
    Console.WriteLine();

    history.AddMessages(updates.ToChatResponse());   // 将回复带入下一轮
}
```

`AGUIChatClient` 按设计**每轮发送完整消息列表**——无状态服务端正是借此看到先前上下文。

## 运行它（完成标准）

1. 启动服务端：在服务端项目中运行 `dotnet run`（例如监听 `:5001`）。
2. 针对它运行客户端。**当客户端增量打印回复时即成功**（多个非空的 `update.Text` 块，而非最后一次性输出一大块）。
3. 直接对服务端做冒烟测试：
   ```bash
   curl -N -X POST http://localhost:5001/ -H "Content-Type: application/json" \
     -d '{"threadId":"t1","runId":"r1","messages":[{"id":"m1","role":"user","content":"Hello"}]}'
   ```
你应该看到 `data: {...}` SSE 帧：`RUN_STARTED` … `TEXT_MESSAGE_CONTENT` … `RUN_FINISHED`。
4. 多轮保持上下文：第二个回答反映第一次交互。

## 托管型代理

当服务端持有对话（以线程 id 为键的会话存储）时使用。服务端从线程 id 回忆历史，因此客户端只需保持该线程 id 稳定——复用同一个 `ChatOptions`（见上文）通常就够了。需要时有两个额外工具：

- **固定已知线程 id**（恢复之前创建的会话）：将其设为 `ConversationId`——`AGUIChatClient` 会将其向内映射为 AG-UI 线程 id。

  ```csharp
  var options = new ChatOptions { ConversationId = threadId };
  ```

- **读取某轮的线程 id / 运行 id**——它们随 `RUN_STARTED` 事件传递，暴露为 update 的 `RawRepresentation`：

  ```csharp
  using AGUI.Abstractions;

  string? threadId = null, runId = null;
  await foreach (var u in client.GetStreamingResponseAsync(messages, options))
  {
      if (u.RawRepresentation is RunStartedEvent started)
      {
          threadId = started.ThreadId;
          runId = started.RunId;
      }
      Console.Write(u.Text);
  }
  ```

- **高级——设置 `ChatOptions` 无对应项的线级字段。** 例如 `ParentRunId`，用于将一次运行链接到上一次。通过 `RawRepresentationFactory` 提供；消息仍通过常规调用参数传递，要只发送新一轮，只需传入更短的列表：

  ```csharp
  var options = new ChatOptions
  {
      RawRepresentationFactory = _ => new RunAgentInput { ParentRunId = runId },
  };

  await foreach (var update in client.GetStreamingResponseAsync(newMessages, options))
  {
      Console.Write(update.Text);
  }
  ```

仅当服务端真正持有历史时，才用这种方式只发送新消息；否则优先使用全历史无状态模式，它适用于任何 AG-UI 服务端。

## 反模式

- **在不同对话间复用同一个 `ChatOptions`。** 线程 id 会固定到选项实例上，因此第二个无关对话会继承第一个的线程 id——面对托管型服务端时，两者会被合并为一个会话。每个对话使用新的 `ChatOptions`；仅在同一对话的各轮之间复用它。
- **想要流式输出时却调用 `GetResponseAsync`。** 它只返回最终聚合响应——没有增量文本，且无内容的更新（生命周期和状态事件）在聚合后不会保留。请使用 `GetStreamingResponseAsync`。