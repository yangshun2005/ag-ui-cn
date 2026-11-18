---
name: agui-dotnet-protobuf
description: >
  在 AG-UI .NET SDK 的 AG-UI 连接中使用 protobuf 传输（替代默认的 Server-Sent Events）——一种通过 Accept 头协商的紧凑二进制事件流。适用场景：通过将配置了 ProtobufEventStreamFormatter（以及作为回退的 SseEventStreamFormatter）的 AGUIEventStreamHandler 接入 HttpClient，使 AGUIChatClient 优先使用 protobuf；通过注册 ProtobufEventStreamFormatter 并根据请求 Accept 头协商响应格式，使服务器能够响应 protobuf；理解 protobuf 或 SSE 的回退机制。不适用场景：默认 SSE 传输或首次设置（请使用 agui-dotnet-streaming-chat）；JSON 事件序列化问题；工具、状态、中断、多模态或生成式 UI。
---

# AG-UI .NET — protobuf 传输

目标：以紧凑的 protobuf 二进制而非 Server-Sent Events 承载 AG-UI 事件流，当一方不支持时自动回退到 SSE。

传输通过内容类型协商：客户端在 `Accept` 头上按优先级声明其接受的格式，服务器以双方都支持的第一个格式回复。传输层之上的一切不变——`AGUIChatClient`、消息和端点逻辑与 SSE 完全相同。

## 安装

```bash
dotnet add package AGUI.Protobuf   # ProtobufEventStreamFormatter
```

客户端应用还需要 `AGUI.Client`（用于 `AGUIEventStreamHandler`）；服务器应用还需要 `AGUI.Formatting`（用于 `SseEventStreamFormatter`）和 `AGUI.Server`。

## 客户端：优先 protobuf，回退到 SSE

将 `HttpClient` 包装在按优先级顺序配置格式化器的 `AGUIEventStreamHandler` 中。该处理器在 `Accept` 上声明这些格式，并解码服务器返回的内容：

```csharp
using AGUI.Client;
using AGUI.Formatting;
using AGUI.Protobuf;
using Microsoft.Extensions.AI;

var handler = new AGUIEventStreamHandler(
    [new ProtobufEventStreamFormatter(), new SseEventStreamFormatter()])
{
    InnerHandler = new HttpClientHandler(),
};

using var httpClient = new HttpClient(handler);
IChatClient client = new AGUIChatClient(new(httpClient, "http://localhost:5013"));
```

Protobuf 列在首位，因此客户端优先使用它；如果服务器只支持 SSE，处理器会透明地改为解码 SSE。切换传输只需重新排序（或裁剪）此格式化器列表。

## 服务器：按需响应 protobuf

注册两个格式化器，并根据 `Accept` 头为每个请求选择一个，默认使用 SSE：

```csharp
using System.Linq;
using AGUI.Abstractions;
using AGUI.Formatting;
using AGUI.Protobuf;
using AGUI.Server;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.Options;
using JsonOptions = Microsoft.AspNetCore.Http.Json.JsonOptions;

builder.Services.AddSingleton<IAGUIEventStreamFormatter, SseEventStreamFormatter>();
builder.Services.AddSingleton<IAGUIEventStreamFormatter, ProtobufEventStreamFormatter>();

var app = builder.Build();

app.MapPost("/", async (
    [FromBody] RunAgentInput input,
    IChatClient chatClient,
    IEnumerable<IAGUIEventStreamFormatter> formatters,
    IOptions<JsonOptions> jsonOptions,
    HttpContext http,
    CancellationToken ct) =>
{
    var accept = http.Request.Headers.Accept.ToString();
    var formatter =
        formatters.FirstOrDefault(f => f is ProtobufEventStreamFormatter
                                       && accept.Contains(f.MediaType, StringComparison.OrdinalIgnoreCase))
        ?? formatters.First(f => f is SseEventStreamFormatter);

    var ctx = input.ToChatRequestContext(jsonOptions.Value.SerializerOptions);
    var updates = chatClient.GetStreamingResponseAsync(ctx.Messages, ctx.ChatOptions, ct);
    var events = updates.AsAGUIEventStreamAsync(ctx, ct);

    http.Response.ContentType = formatter.MediaType;
    http.Response.Headers.CacheControl = "no-cache";
    await formatter.WriteAsync(events, http.Response.Body, ct);
});
```

仅当客户端*显式*接受 protobuf 媒体类型时才选择 protobuf，因此默认的 SSE 客户端无需改动即可继续工作。

## 反模式

- **仅在单侧注册 protobuf 格式化器。** 协商需要两端一致。如果服务器注册了 protobuf 但客户端处理器只列出 SSE（或反之），每次连接都会静默回退到 SSE，二进制路径永远不会被使用——请验证实际选中的 `Content-Type`，不要假设。
- **手动解析 protobuf 正文。** 线上帧格式必须与 `@ag-ui/proto` 逐字节一致。始终通过 `ProtobufEventStreamFormatter` 读写；不要自行解码流。

## 验证

1. 两端都配置后，响应的 `Content-Type` 为 protobuf 媒体类型（`ProtobufEventStreamFormatter.MediaType`），且正文为二进制，而非 `data:` 文本帧。
2. 将同一客户端指向仅支持 SSE 的服务器（移除服务器的 protobuf 注册）：它仍能工作，此时通过 SSE——确认回退生效。
3. 解码后的事件流和助手输出与 SSE 传输完全相同。