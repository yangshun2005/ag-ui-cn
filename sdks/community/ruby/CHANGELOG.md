# 更新日志

AG-UI Ruby SDK 的所有重要变更都记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)，
本项目遵循 [语义化版本](https://semver.org/spec/v2.0.0.html)。

## [未发布]

### 新增

- [`RunStartedEvent`](https://docs.ag-ui.com/spec/draft/schema#runstartedevent)、[`RunAgentInput`](https://docs.ag-ui.com/spec/draft/schema#runagentinput) 可按规范定义，通过可选的 `protocol_version` 参数创建。

## [0.2.0] - 2026-05-26

### 新增

- 推理事件：`ReasoningStartEvent`、`ReasoningEndEvent`、`ReasoningMessageStartEvent`、`ReasoningMessageContentEvent`、`ReasoningMessageEndEvent`、`ReasoningMessageChunkEvent`、`ReasoningEncryptedValueEvent`
- `RunFinishedEvent` 支持运行结果：`RunFinishedSuccessOutcome`、`RunFinishedInterruptOutcome`
- `TEXT_MESSAGE_ROLE_VALUES` 新增 `reasoning` 角色
- 能力模块（`AgUiProtocol::Core::Capabilities`），包含智能体能力声明：
  - `IdentityCapabilities` — 智能体名称、类型、描述、版本、提供方、文档 URL 和元数据
  - `TransportCapabilities` — 支持的传输方式（streaming、websocket、http_binary、push_notifications、resumable）
  - `ToolsCapabilities` — 工具调用支持、智能体提供的工具、并行调用、客户端提供的工具
  - `OutputCapabilities` — 结构化输出和支持的 MIME 类型
  - `StateCapabilities` — 状态快照、增量、长期记忆、持久状态
  - `MultiAgentCapabilities` — 多智能体协调、委派、交接、子智能体
  - `ReasoningCapabilities` — 推理/思考支持、流式传输、加密推理
  - `MultimodalInputCapabilities` — 图像、音频、视频、PDF、文件输入
  - `MultimodalOutputCapabilities` — 图像和音频输出
  - `MultimodalCapabilities` — 输入/输出多模态组合支持
  - `ExecutionCapabilities` — 代码执行、沙箱、迭代和时间限制
  - `HumanInTheLoopCapabilities` — 审批、干预、反馈、中断协议、带编辑审批
  - `AgentCapabilities` — 聚合上述所有能力的分类快照
  - `SubAgentInfo` — 父智能体调用的子智能体描述符

### 变更

- 用于生成 Markdown 文档的 YARD 模板配置
- 更新 Ruby SDK 文档页面（概览、类型、事件、能力）

## [0.1.5] - 2026-02-11

- 更新 gemspec 中的文档和更新日志 URL

## [0.1.4] - 2026-01-05

- 新增 `ag-ui-protocol.rb` 作为 `ag_ui_protocol.rb` 的别名，并修复 tapioca rbi 生成 bug
- 移除冗余的 Sorbet 类型 `T.nilable(T.untyped)`，修复 Sorbet 警告

## [0.1.0] - 2025-12-18

### 新增

- AG-UI Ruby SDK 首次发布
- 核心协议实现，包含强类型模型（`AgUiProtocol::Core::Types`）
- 完整事件类型支持（`AgUiProtocol::Core::Events`）
- 通过 `AgUiProtocol::EventEncoder` 实现 Server-Sent Events (SSE) 编码
- 自动 camelCase JSON 序列化并移除 `nil` 值
- 通过 `sorbet-runtime` 进行运行时校验
- 覆盖类型、事件和编码的测试套件

[0.2.0]: https://github.com/ag-ui-protocol/ag-ui/releases/tag/sdk%2Fruby%2Fv0.2.0
[0.1.5]: https://github.com/ag-ui-protocol/ag-ui/releases/tag/sdk%2Fruby%2Fv0.1.5
[0.1.4]: https://github.com/ag-ui-protocol/ag-ui/releases/tag/sdk%2Fruby%2Fv0.1.4
[0.1.0]: https://github.com/ag-ui-protocol/ag-ui/tree/main/sdks/community/ruby