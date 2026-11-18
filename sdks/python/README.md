# ag-ui-protocol

**Agent-User Interaction (AG-UI) 协议**的 Python SDK。

`ag-ui-protocol` 为 Python 开发者提供强类型数据结构和事件编码，用于构建兼容 AG-UI 的 Agent 服务器。基于 Pydantic 实现可靠校验，并自动进行 camelCase 序列化，无缝对接前端。

## 安装

```bash
pip install ag-ui-protocol
poetry add ag-ui-protocol
pipenv install ag-ui-protocol
```

## 特性

- 🐍 **Python 原生** – 符合 Python 习惯的 API，完整类型提示与校验
- 📋 **Pydantic 模型** – 运行时校验与自动 JSON 序列化
- 🔄 **流式事件** – 16 种核心事件类型，支持实时 Agent 通信
- ⚡ **高性能** – 高效的 Server-Sent Events 事件编码

## 快速示例

```python
from ag_ui.core import TextMessageContentEvent, EventType
from ag_ui.encoder import EventEncoder

# Create a streaming text event
event = TextMessageContentEvent(
    type=EventType.TEXT_MESSAGE_CONTENT,
    message_id="msg_123",
    delta="Hello from Python!"
)

# Encode for HTTP streaming
encoder = EventEncoder()
sse_data = encoder.encode(event)
# Output: data: {"type":"TEXT_MESSAGE_CONTENT","messageId":"msg_123","delta":"Hello from Python!"}\n\n
```

### 多模态用户消息

```python
from ag_ui.core import UserMessage, TextInputContent, ImageInputPart, InputContentUrlSource

message = UserMessage(
    id="user-123",
    content=[
        TextInputContent(text="Please describe this image"),
        ImageInputPart(
            source=InputContentUrlSource(
                value="https://example.com/cat.png",
                mime_type="image/png",
            )
        ),
    ],
)

payload = message.model_dump(by_alias=True)
# {"id": "user-123", "role": "user", "content": [...]}
```

> `BinaryInputContent` 已弃用。请改用模态专属的输入部件（`ImageInputPart`、`AudioInputPart`、`VideoInputPart`、`DocumentInputPart`），并搭配 `InputContentDataSource` 或 `InputContentUrlSource`。

### 无值的可选字段会被省略

序列化 AG-UI 类型时，所有无值的可选字段都会被省略，而不会写成 `null`——与 TypeScript 生产者发送到线上的内容一致。该行为内置于基类模型，因此在所有路径上都生效（`model_dump`、`model_dump_json`、嵌套在其他模型中、`EventEncoder`），无需传入 `exclude_none=True`：

```python
from ag_ui.core import ToolCallStartEvent

ToolCallStartEvent(tool_call_id="tc_1", tool_call_name="search").model_dump_json(by_alias=True)
# {"type":"TOOL_CALL_START","toolCallId":"tc_1","toolCallName":"search"}
# note: no "parentMessageId": null
```

作为实际值的 `null` 不受影响：持有 `None` 的必填字段、`dict` 或 `list` 中的 `None`（单个元数据值、带 `null` 的 JSON Patch `replace`），以及任何额外字段，都会序列化为 `null`。

## 包结构

- **`ag_ui.core`** – AG-UI 协议的类型、事件和数据模型
- **`ag_ui.encoder`** – HTTP 流式传输的事件编码工具

## 文档

- 概念与架构：[`docs/concepts`](https://docs.ag-ui.com/concepts/architecture)
- 完整 API 参考：[`docs/sdk/python`](https://docs.ag-ui.com/sdk/python/core/overview)

## 贡献

欢迎提交 Bug 报告和 Pull Request！请先阅读我们的[贡献指南](https://docs.ag-ui.com/development/contributing)。

## 许可证

MIT © 2025 AG-UI Protocol Contributors