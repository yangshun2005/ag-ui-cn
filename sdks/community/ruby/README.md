# ag-ui-protocol

**Agent-User Interaction (AG-UI) 协议**的 Ruby SDK。

`ag-ui-protocol` 为 Ruby 开发者提供强类型数据结构和事件编码，用于构建兼容 AG-UI 的 agent 服务器。基于 Sorbet 实现健壮的校验，并自动进行 camelCase 序列化，无缝对接前端。

## 安装

安装 bundler：

```bash
gem install bundler
```

添加 gem：

```bash
bundle add ag-ui-protocol
```

## 特性

- 🐍 **Ruby 原生** – Ruby API，带完整类型提示和校验
- 📋 **Sorbet 运行时** – 运行时校验方案
- 🔄 **流式事件** – 16 种核心事件类型，支持实时 agent 通信
- ⚡ **高性能** – 为 Server-Sent Events 提供高效事件编码

## 快速示例

```ruby
require "ag_ui_protocol"

event = AgUiProtocol::Core::Events::TextMessageContentEvent.new(
    message_id: "msg_123",
    delta: "Hello from Ruby!",
)

encoder = AgUiProtocol::Encoder::EventEncoder.new
encoded_event = encoder.encode(event)
```

### 多模态用户消息

```ruby
require "ag_ui_protocol/core/types"

message = AgUiProtocol::Core::Types::UserMessage.new(
    id: "user-123",
    content: [
        { type: "text", text: "Please describe this image" },
        { type: "binary", mimeType: "image/png", url: "https://example.com/a.png" }
        # or
        AgUiProtocol::Core::Types::TextInputContent.new(text: "Please describe this image"),
        AgUiProtocol::Core::Types::BinaryInputContent.new(mime_type: "image/png", url: "https://example.com/cat.png"),
    ],
)
```

## 包结构

- **`AgUiProtocol::Core::Types`** – 消息类型、工具和数据模型
- **`AgUiProtocol::Core::Events`** – 事件类型和事件处理
- **`AgUiProtocol::Encoder`** – 用于 HTTP 流式传输的事件编码工具

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/ruby`](../../../docs/sdk/ruby/overview.mdx)

## 示例

参见 [`example/`](example/) 目录：

- [简单用例](example/simple-use/README.md)
- [最小 Rails 示例](example/rails/README.md)

## 同步文档

要将 YARD 文档同步到 `docs/sdk/ruby` 路径，运行以下命令：

```bash
cd sdks/community/ruby
rake doc
```

## 测试

```bash
cd sdks/community/ruby
rake test
```

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读我们的[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors