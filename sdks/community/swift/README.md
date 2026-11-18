# AGUISwift

[![CI](https://github.com/paduh/ag-ui/actions/workflows/ci.yml/badge.svg)](https://github.com/paduh/ag-ui/actions/workflows/ci.yml)

AG-UI Swift SDK 是一个 Swift 库，用于构建实现 Agent User Interaction Protocol (AG-UI) 的 AI 智能体用户界面。它提供 Swift 应用与 AI 智能体之间的实时流式通信。

## 协议兼容性

本 SDK 面向 1.0 之前的 AG-UI 契约：基于 HTTP/SSE 的生命周期、文本、工具调用、状态快照/增量以及旧版思考事件。它不声称完全符合 AG-UI 1.0 Schema 或流式规范。特别是，1.0 多模态内容结构和完整的 1.0 事件覆盖不作保证；未知或不支持的事件遵循配置的解码器策略。完整的 1.0 支持和一致性验证将另行跟进。

## 架构

SDK 采用模块化架构，包含四个主要组件：

### AGUIAgentSDK
常见智能体交互模式的高层 API。
- **AgUiAgent**：无状态客户端，适用于无需持续上下文或智能体在服务端管理所有状态的场景
- **StatefulAgUiAgent**：有状态客户端，维护对话历史并在每次请求时发送
- **Builders**：便捷的智能体配置构建器模式

### AGUIClient
底层客户端基础设施和传输实现。
- **HttpAgent**：底层 HTTP 传输实现
- **AbstractAgent**：自定义智能体实现的基类
- **SseParser**：用于流式响应的 Server-Sent Events 解析器
- **EventStreamManager**：事件流管理与处理

#### HTTP 流式响应

`HttpTransport` 将配置的请求头应用于默认客户端和注入的 `HTTPClient` 实现。显式请求头会覆盖默认的 `Accept` 和 `Content-Type` 值；提供的恢复游标会覆盖配置的 `Last-Event-ID` 请求头。

流式运行必须返回 2xx 状态且 `Content-Type: text/event-stream`。媒体类型匹配忽略大小写，并接受 `charset=utf-8` 等参数。非 2xx 响应抛出 `ClientError.httpError(statusCode:)`；媒体类型缺失或不正确抛出 `ClientError.invalidResponse`。

### AGUICore
协议类型、事件和消息定义。
- **Events**：所有 AG-UI 协议事件类型及序列化
- **Types**：协议消息类型和状态管理
- **领域层**：纯领域值对象和领域事件
- **基础设施层**：序列化和适配器

### AGUITools
用于扩展智能体能力的工具执行框架。
- **ToolExecutor**：实现自定义工具的协议
- **ToolRegistry**：工具注册与管理
- **ToolExecutionManager**：带熔断器模式的工具执行

## 环境要求

- Swift 5.9+
- iOS 15.0+ / macOS 13.0+

## 文档

可使用 Swift-DocC 插件在本地生成 API 文档：

```bash
swift package generate-documentation
```

## 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。


## 贡献

欢迎贡献！请遵循我们的开发流程：

### 快速开始

1. **创建功能分支**：
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **修改并测试**：
   ```bash
   swift test
   swiftlint lint
   ```

4. **创建 Pull Request**：
   ```bash
   git push -u origin feature/your-feature-name
   gh pr create
   ```

### 开发流程

- **禁止直接提交到 `main`** - 所有更改必须通过 Pull Request
- **需要代码审查** - PR 合并前必须获得指定审查者的批准
- **分支命名**：`feature/`、`fix/`、`refactor/`、`docs/`
- **遵循 TDD**：先写测试，再实现
- **代码质量**：使用 SwiftFormat 格式化，使用 SwiftLint 检查

详见指南：
- [贡献指南](CONTRIBUTING.md) - 完整流程和标准
- [工作流快速参考](docs/WORKFLOW.md) - 常见任务和命令
- [项目标准](CLAUDE.md) - 代码约定和架构

### 提交 PR 前

- [ ] 测试通过：`swift test`
- [ ] 代码已格式化：`swift package plugin swiftformat`
- [ ] Lint 检查通过：`swiftlint lint`
- [ ] 文档已更新
- [ ] 提交信息遵循约定