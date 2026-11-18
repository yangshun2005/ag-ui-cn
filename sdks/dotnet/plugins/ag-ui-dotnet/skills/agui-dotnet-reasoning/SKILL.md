---
name: agui-dotnet-reasoning
description: >
  使用 AG-UI .NET SDK 将推理/思考模型的中间思考过程与其最终答案分开呈现——让客户端能在 agent 响应前显示其"思考"过程，并在多轮对话中正确处理该推理轨迹。适用场景：在客户端通过匹配每个 ChatResponseUpdate 的 Contents 中的 TextReasoningContent 与 TextContent，区分模型的推理轨迹与答案；在流式输出时以不同方式渲染思考与答案；决定是否将推理文本带入后续轮次（是否将模型的推理追加/重发回对话历史）。不适用场景：无推理展示的纯文本流式输出（请使用 agui-dotnet-streaming-chat）；工具、共享状态、中断、多模态、生成式 UI 或 protobuf。
---

# AG-UI .NET — 推理（思考）事件

目标：在推理模型的中间思考流式输出时予以展示，并与最终答案在视觉上分开。

推理模型在流中输出两类文本：思考与答案。在 `Microsoft.Extensions.AI` 中，它们以不同的内容类型出现在每个 `ChatResponseUpdate` 上——思考为 `TextReasoningContent`，答案为 `TextContent`。`update.Text` 只拼接答案部分，因此要显示思考需检查 `update.Contents`。

## 将思考与答案分开

遍历 `update.Contents` 并按内容类型分支，而非读取 `update.Text`：

```csharp
using Microsoft.Extensions.AI;

var messages = new List<ChatMessage>
{
    new(ChatRole.User, "20 heads, 56 legs — how many chickens and rabbits? Show your reasoning."),
};

await foreach (var update in client.GetStreamingResponseAsync(messages))
{
    foreach (var content in update.Contents)
    {
        switch (content)
        {
            case TextReasoningContent { Text: { Length: > 0 } thinking }:
                RenderThinking(thinking);   // e.g. a dimmed/collapsible panel
                break;
            case TextContent { Text: { Length: > 0 } answer }:
                RenderAnswer(answer);       // the user-facing reply
                break;
        }
    }
}
```

两者都增量流式输出且可交错：先到思考增量，再到答案增量。应随到随渲染而非缓冲，这样在答案到达前即可看到"思考中…"状态。

## 反模式

- **将推理文本作为对话历史重发。** 思考是单轮的草稿，不是消息。将捕获的 `TextReasoningContent` 追加到下一次请求的 messages 中会膨胀提示词，且部分提供方会拒绝将先前的推理作为输入。只携带助手的答案。
- **将推理视为答案的一部分。** 思考是独立通道；将其拼入回复会让用户把模型的草稿当作响应。保持两条渲染路径独立。

## 验证

1. 对于要求模型推理的提示词，流会在最终 `TextContent` 答案之前产出 `TextReasoningContent` 更新。
2. 渲染的答案只包含响应内容，推理通过其自身路径显示（或隐藏）——不混入答案文本。