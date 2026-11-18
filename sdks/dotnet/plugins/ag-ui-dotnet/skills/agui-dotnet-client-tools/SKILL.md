---
name: agui-dotnet-client-tools
description: >
  使用 AG-UI .NET SDK 向 AG-UI 智能体暴露客户端（前端）工具——即在客户端应用中运行的 C# 函数（读取本地状态、GPS、UI、设备 API），由客户端执行调用并返回结果，使运行继续。适用场景：在客户端声明 AIFunction 并通过 ChatOptions.Tools 传给 AGUIChatClient；让客户端执行模型请求的工具并自动回传结果；理解客户端为何无需 UseFunctionInvocation（AGUIChatClient 已调用函数）；服务端必须做什么（UseFunctionInvocation + TerminateOnUnknownCalls）才能转发未知/客户端工具而非报错。不适用场景：在服务端/后端运行的工具（使用 agui-dotnet-server-tools）；执行前暂停等待人工批准/输入（中断 / 人在回路）；普通聊天（使用 agui-dotnet-streaming-chat）；共享状态、生成式 UI、多模态或 protobuf。
---

# AG-UI .NET — 客户端（前端）工具

目标：让模型调用在客户端应用中运行的函数——读取只有客户端才有的信息（位置、本地状态、UI 选中项、设备 API）——由客户端执行调用并返回结果，使运行继续直至最终答案。

客户端工具就是一个普通的 `Microsoft.Extensions.AI` `AIFunction`，通过 `ChatOptions.Tools` 附加到请求上。`AGUIChatClient` 本身就是会调用函数的客户端，因此它在本地运行工具并自动回传结果——无需再用 `UseFunctionInvocation` 包装。

## 安装

```bash
dotnet add package AGUI.Client
```

`Microsoft.Extensions.AI` 提供 `AIFunctionFactory`、`ChatOptions` 和 `AITool`。运行 `dotnet package search AGUI.Client --exact-match` 查看当前版本。

## 声明工具并在选项中传入

在客户端定义函数，用 `AIFunctionFactory.Create` 包装，放入 `ChatOptions.Tools`：

```csharp
using System.ComponentModel;
using AGUI.Client;
using Microsoft.Extensions.AI;

[Description("Get the user's current location from GPS.")]
static string GetUserLocation() => "Amsterdam, Netherlands (52.37°N, 4.90°E)";

using var httpClient = new HttpClient();
IChatClient client = new AGUIChatClient(new(httpClient, "http://localhost:5003"));

var options = new ChatOptions { Tools = [AIFunctionFactory.Create(GetUserLocation)] };

var messages = new List<ChatMessage> { new(ChatRole.User, "What's fun to do near me?") };

await foreach (var update in client.GetStreamingResponseAsync(messages, options))
{
    Console.Write(update.Text);
}
```

当模型决定调用 `GetUserLocation` 时，`AGUIChatClient` 在本机调用它，将结果返回服务端，运行继续——你打印的流式文本已反映该位置。

## 服务端必须做什么

服务端将不属于自己的工具转发给客户端，而不是尝试运行它。为服务端的聊天客户端注册函数调用并启用 `TerminateOnUnknownCalls`，使未知（客户端）工具干净地结束服务端回合，并交由客户端执行：

```csharp
builder.Services.AddChatClient(/* provider IChatClient */)
    .UseFunctionInvocation(fic => fic.TerminateOnUnknownCalls = true);
```

服务端不声明任何客户端工具；只需转发它们。

## 反模式

- **用 `UseFunctionInvocation` 包装客户端。** `AGUIChatClient` 已调用函数；再加一层函数调用会重复处理调用，可能导致执行两次或破坏 AG-UI 工具路由。通过 `ChatOptions.Tools` 传入工具，让客户端调用它。
- **客户端工具在其报告的副作用完成前就返回。** 函数一返回模型就继续运行，因此若工具声称"已保存"而异步写入仍在进行，模型会叙述尚未发生的结果。在函数内等待实际工作完成后再返回。

## 验证

1. 运行一个模型只能借助客户端工具回答的提示。函数体在本地执行（在其中设断点或打日志）。
2. 流式答案反映工具的返回值，且运行在单次客户端调用中以 `RUN_FINISHED` 结束——你无需编写任何代码回传工具结果。