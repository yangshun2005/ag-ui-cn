---
name: agui-dotnet-multimodal
description: >
  使用 AG-UI .NET SDK 向 AG-UI 智能体发送图片及其他二进制/文件内容——将图片（或音频、PDF 等）附加到用户消息，使多模态模型能够看到它们。适用场景：构建包含混合内容部分（TextContent 加用于内联字节的 DataContent，或用于托管 URL 的 UriContent）的用户 ChatMessage；选择内联字节还是 URL 引用；设置正确的媒体类型；通过 AGUIChatClient 发送消息，使内容部分经 AG-UI 协议传输至视觉/多模态模型。不适用场景：纯文本聊天（使用 agui-dotnet-streaming-chat）；工具调用（使用 agui-dotnet-server-tools / agui-dotnet-client-tools）；结构化共享状态（使用 agui-dotnet-shared-state）；推理轨迹、中断、生成式 UI 或 protobuf。
---

# AG-UI .NET — 多模态消息

目标：在一轮对话中包含图片（或其他二进制内容），使多模态模型能够描述或推理它。

多模态消息就是一个普通的 `Microsoft.Extensions.AI` `ChatMessage`，其 `Contents` 混合了一个 `TextContent` 和一个或多个二进制部分。`AGUIChatClient` 将这些部分经 AG-UI 协议传输；服务器照常流式返回模型回复。

## 安装

```bash
dotnet add package AGUI.Client
```

`Microsoft.Extensions.AI` 提供 `ChatMessage`、`TextContent`、`DataContent` 和 `UriContent`。

## 附加内联字节

使用 `DataContent(bytes, mediaType)` 将数据直接嵌入消息：

```csharp
using AGUI.Client;
using Microsoft.Extensions.AI;

byte[] imageBytes = await File.ReadAllBytesAsync("photo.png");

var messages = new List<ChatMessage>
{
    new(ChatRole.User,
    [
        new TextContent("Describe this image"),
        new DataContent(imageBytes, "image/png"),
    ]),
};

await foreach (var update in client.GetStreamingResponseAsync(messages))
{
    Console.Write(update.Text);
}
```

媒体类型必须与字节匹配（`image/png`、`image/jpeg`、`audio/wav`、`application/pdf` 等），以便模型正确解码。

## 引用托管 URL

当资源已存在于某个 URL 时，改用 `UriContent(uri, mediaType)`——字节不经过你的进程：

```csharp
new ChatMessage(ChatRole.User,
[
    new TextContent("What's in this picture?"),
    new UriContent("https://example.com/cat.jpg", "image/jpeg"),
]);
```

优先使用 `UriContent` 引用已托管在 URL 的资源；对于模型主机无法通过 URL 访问的本地字节，使用 `DataContent`。

## 反模式

- **将大型资源作为内联 `DataContent` 发送。** 内联二进制会被 base64 编码进请求，膨胀约 33%，并迫使整个载荷在两端经过内存。对于任何可通过 URL 访问的较大资源，应发送 `UriContent`，让模型主机自行获取。
- **面向非多模态模型。** 内容部分可以正常传输，但纯文本部署会忽略或拒绝图片。应将服务器的 `IChatClient` 指向支持视觉的模型/部署。

## 验证

1. 助手的流式回复描述了实际的图片内容（而非通用的"我无法查看图片"消息）。
2. 请求携带了二进制部分：传输中的用户消息同时包含文本部分和图片/数据部分，且媒体类型为你设置的值。