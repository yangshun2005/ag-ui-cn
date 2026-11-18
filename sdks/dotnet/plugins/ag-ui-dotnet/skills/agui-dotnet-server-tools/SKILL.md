---
name: agui-dotnet-server-tools
description: >
  使用 AG-UI .NET SDK 暴露 AG-UI 代理可调用的服务端（后端）工具——在服务器上运行的 C# 函数，由服务器执行调用并将结果回传给模型。适用场景：用 AIFunctionFactory.Create 定义 AIFunction，并通过 ConfigureOptions + UseFunctionInvocation 将其注册到服务器的 IChatClient；让模型在运行期间调用你的后端函数；AOT 安全的工具参数/结果（为复杂工具参数类型注册 JsonSerializerContext）；并行/并发后端工具调用（AllowConcurrentInvocation）；TerminateOnUnknownCalls 行为。不适用场景：在客户端/前端运行的工具（请用 agui-dotnet-client-tools）；暂停工具等待人工批准或输入（中断 / 人在回路）；无工具的普通流式聊天（请用 agui-dotnet-streaming-chat）；共享状态、生成式 UI、多模态或 protobuf。
---

# AG-UI .NET — 服务端（后端）工具

目标：让模型调用在服务器上运行的 C# 函数，由服务器执行调用并返回结果，使运行继续直至最终答案。

服务端工具就是一个注册在服务器 `IChatClient` 上的普通 `Microsoft.Extensions.AI` `AIFunction`。`FunctionInvokingChatClient` 在运行内部执行它；客户端不发送任何工具、不需要工具代码，只接收最终答案。工具调用和结果事件仍会经过网络传输——只是客户端从不执行任何东西。

## 安装

```bash
dotnet add package AGUI.Server
dotnet add package AGUI.Formatting
```

`Microsoft.Extensions.AI` 提供 `AIFunctionFactory`、`AddChatClient` 和 `UseFunctionInvocation`。运行 `dotnet package search AGUI.Server --exact-match` 查看当前版本。

## 定义并注册工具

用 `AIFunctionFactory.Create` 创建函数，将其加入聊天客户端的工具列表，并启用函数调用：

```csharp
using System.ComponentModel;
using Microsoft.Extensions.AI;

[Description("Search for restaurants in a location.")]
static RestaurantSearchResponse SearchRestaurants(
    [Description("Where to search and what cuisine.")] RestaurantSearchRequest request)
{
    // ... real lookup ...
}

var builder = WebApplication.CreateBuilder(args);

var searchRestaurants = AIFunctionFactory.Create(
    SearchRestaurants,
    serializerOptions: SampleJsonSerializerContext.Default.Options);

builder.Services.AddChatClient(
        new AzureOpenAIClient(new Uri(endpoint), new DefaultAzureCredential())
            .GetChatClient(deploymentName)
            .AsIChatClient())
    .ConfigureOptions(o => (o.Tools ??= []).Add(searchRestaurants))
    .UseFunctionInvocation(fic => fic.TerminateOnUnknownCalls = true);
```

`[Description]` 特性会成为模型看到的工具和参数 Schema。任何 `Microsoft.Extensions.AI` 提供程序都可替代 Azure OpenAI。

## AOT 安全的工具参数和结果

当工具接收或返回复杂类型（任何非基元类型）时，其 Schema 和（反）序列化必须由源生成，而非基于反射。将参数和结果类型放入 `JsonSerializerContext`，注册到主机的 JSON 选项，并传给 `AIFunctionFactory.Create`：

```csharp
[JsonSerializable(typeof(RestaurantSearchRequest))]
[JsonSerializable(typeof(RestaurantSearchResponse))]
internal sealed partial class SampleJsonSerializerContext : JsonSerializerContext;

builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.TypeInfoResolverChain.Add(SampleJsonSerializerContext.Default));

var searchRestaurants = AIFunctionFactory.Create(
    SearchRestaurants,
    serializerOptions: SampleJsonSerializerContext.Default.Options);
```

同一个上下文既注册到主机（使网络传输的 `RunAgentInput` 能往返这些类型），也传给函数（使其参数绑定使用源生成）。

## 托管端点

端点流式传输已注册的 `IChatClient`——函数调用在流内部由服务端解析，之后文本才到达客户端：

```csharp
using AGUI.Abstractions;
using AGUI.Formatting;
using AGUI.Server;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.Options;
using JsonOptions = Microsoft.AspNetCore.Http.Json.JsonOptions;

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
    var ctx = input.ToChatRequestContext(jsonOptions.Value.SerializerOptions);
    var updates = chatClient.GetStreamingResponseAsync(ctx.Messages, ctx.ChatOptions, ct);
    var events = updates.AsAGUIEventStreamAsync(ctx, ct);

    http.Response.ContentType = formatter.MediaType;
    http.Response.Headers.CacheControl = "no-cache";
    await formatter.WriteAsync(events, http.Response.Body, ct);
});

app.Run();
```

## 并行工具调用

当模型在一轮中请求多个工具调用时，让函数调用客户端并发执行它们：

```csharp
.UseFunctionInvocation(fic =>
{
    fic.TerminateOnUnknownCalls = true;
    fic.AllowConcurrentInvocation = true;
});
```

模型决定是否批量调用；`AllowConcurrentInvocation` 只控制已请求的调用是并行执行还是串行执行。

## 反模式

- **注册带复杂参数的工具但没有源生成上下文。** 基于反射的 Schema 生成在 Native AOT 和裁剪下会失效。每个非基元工具参数或结果类型都需要一个 `JsonSerializerContext` 条目，注册到主机并传给 `AIFunctionFactory.Create`。
- **同时涉及客户端工具时让 `TerminateOnUnknownCalls` 保持默认值。** 设置后，服务器不拥有的工具（客户端声明的工具）会干净地结束服务端运行，以便客户端执行它；不设置则未知调用会在运行中途以错误形式暴露。

## 验证

1. 发送一个需要该工具的提示并观察流：`TOOL_CALL_START` / `TOOL_CALL_ARGS` / `TOOL_CALL_END`，随后是 `TOOL_CALL_RESULT`，然后是使用该结果的助手文本——全部在一次以 `RUN_FINISHED` 结束的运行内完成。
2. 客户端收到最终答案，自身没有任何工具代码。
3. 若面向 Native AOT，`dotnet publish` 对工具类型不产生裁剪/AOT 警告。