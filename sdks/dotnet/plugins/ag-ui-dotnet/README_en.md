# ag-ui-dotnet

用于构建 [AG-UI .NET SDK](https://docs.ag-ui.com) 应用的 Agent 技能。

该插件提供按需加载的、面向具体任务的技能，供 AI 编码 Agent 使用：

| 技能 | 用途 |
| --- | --- |
| `agui-dotnet-streaming-chat` | 搭建客户端 + 服务端并流式传输多轮对话；无状态 Agent 与托管 Agent |
| `agui-dotnet-server-tools` | 暴露 Agent 在服务端调用的后端 C# 函数 |
| `agui-dotnet-client-tools` | 暴露 Agent 在应用中调用的客户端函数 |
| `agui-dotnet-human-in-the-loop` | 让工具需经审批才能执行，或暂停等待用户输入后再恢复 |
| `agui-dotnet-shared-state` | 在 Agent 与客户端之间同步结构化状态（快照/增量） |
| `agui-dotnet-multimodal` | 向多模态模型发送图片及其他二进制内容 |
| `agui-dotnet-reasoning` | 将推理模型的思考过程与最终答案分开呈现 |
| `agui-dotnet-protobuf` | 使用 protobuf 传输协议替代 SSE |
| `agui-dotnet-troubleshoot` | 诊断常见的 AG-UI .NET 问题 |

## 安装

该插件随仓库根目录下的 `ag-ui` 市场一同发布。

### GitHub Copilot CLI

```shell
copilot plugin marketplace add ag-ui-protocol/ag-ui
copilot plugin install ag-ui-dotnet@ag-ui
```

### Claude Code

```shell
/plugin marketplace add ag-ui-protocol/ag-ui
/plugin install ag-ui-dotnet@ag-ui
```

如需直接从本地检出安装，可将上述工具指向本目录（例如 `copilot plugin install ./sdks/dotnet/plugins/ag-ui-dotnet`）。