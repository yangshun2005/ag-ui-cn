# create-ag-ui-app

用于搭建 **Agent-User Interaction (AG-UI) Protocol** 应用的 CLI 工具。

`create-ag-ui-app` 提供交互式安装向导，帮助你按偏好的客户端框架和 Agent 后端快速初始化 AG-UI 项目。

可选 CopilotKit/Next.js Web 应用，或基于终端的 CLI 客户端。

## 用法

```bash
npx create-ag-ui-app@latest
pnpx create-ag-ui-app@latest
bunx create-ag-ui-app@latest
```

## 特性

- 🎯 **交互式安装** – 引导选择客户端和框架
- 🌐 **多种客户端** – CopilotKit/Next.js Web 应用和 CLI 客户端
- 🔧 **框架集成** – 内置支持 LangGraph、CrewAI、Mastra、Agno、LlamaIndex 等
- 📦 **零配置** – 自动配置依赖和项目结构
- ⚡ **快速上手** – 几分钟内从想法到可运行应用

## 快速示例

```bash
# 交互式安装
npx create-ag-ui-app@latest

# 使用框架参数
npx create-ag-ui-app@latest --langgraph-py
npx create-ag-ui-app@latest --mastra

# 查看所有选项
npx create-ag-ui-app@latest --help
```

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/events`](https://docs.ag-ui.com/concepts/events)
- Google ADK JavaScript 快速入门：[docs.ag-ui.com/integrations/google-adk-js](https://docs.ag-ui.com/integrations/google-adk-js)

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读[贡献指南](https://github.com/ag-ui-protocol/ag-ui/blob/main/CONTRIBUTING.md)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors