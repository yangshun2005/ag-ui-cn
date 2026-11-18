# AG-UI .NET SDK

AG-UI（Agent User Interaction Protocol）标准化 AI 智能体与用户界面之间的流式通信。.NET SDK 提供协议类型、传输格式器、protobuf 支持、HTTP 客户端，以及基于 `Microsoft.Extensions.AI` 构建的框架无关服务端适配器。

## 包

- `AGUI.Abstractions` — 协议事件、消息、工具、能力及 JSON 序列化。
- `AGUI.Formatting` — 事件流格式器抽象及 Server-Sent Events 支持。
- `AGUI.Protobuf` — protobuf 编解码器及二进制事件流格式化。
- `AGUI.Client` — AG-UI HTTP 客户端及 `IChatClient` 集成。
- `AGUI.Server` — 服务端适配器，将 `ChatResponseUpdate` 流转换为 AG-UI 事件。

文档、示例及源码见 https://github.com/ag-ui-protocol/ag-ui。