# 更新日志

ag-ui-4k 的所有重要变更都将记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)，
本项目遵循 [语义化版本](https://semver.org/spec/v2.0.0.html)。

## [未发布]

## [0.4.1] - 2026-06-02

### 新增
- **中断（Interrupts）**（[AG-UI 规范](https://docs.ag-ui.com/concepts/interrupts)）。Kotlin SDK 现已实现 TypeScript 和 Python SDK 已支持的中断协议。若无此变更，连接到支持中断的服务端的 Kotlin 客户端要么在 `outcome` 的多态反序列化时失败，要么在 `RUN_FINISHED` 事件中静默丢弃中断载荷。
  - `com.agui.core.types` 中的新类型：
    - `Interrupt(id, reason, message?, toolCallId?, responseSchema?, expiresAt?, metadata?)`
    - `ResumeStatus` 枚举（`RESOLVED` → `"resolved"`，`CANCELLED` → `"cancelled"`）
    - `ResumeEntry(interruptId, status, payload?)`
    - 密封类 `RunFinishedOutcome`，带 `@JsonClassDiscriminator("type")`：
      - `RunFinishedSuccessOutcome`（`{"type":"success"}`）
      - `RunFinishedInterruptOutcome(interrupts)`（`{"type":"interrupt","interrupts":[…]}`）——`interrupts` 在构造时校验非空。
  - `RunAgentInput` 新增可选字段 `resume: List<ResumeEntry>?`，用于在同一 `threadId` 上恢复先前被中断的运行。
  - `RunFinishedEvent` 新增可选字段 `result: JsonElement?` 和 `outcome: RunFinishedOutcome?`。两者默认均为 `null`；省略它们的旧版生产者仍可正常解码，而 Python `exclude_none=False` 调用方显式输出 JSON `null` 时也解码为 `null`。
  - `AgUiSerializersModule` 注册了两个 `RunFinishedOutcome` 子类以支持多态序列化。
  - `InterruptSerializationTest` 中新增 13 个测试，覆盖最小/完整 `Interrupt` 往返、`ResumeEntry` 状态枚举映射及未知状态拒绝、对象载荷、`RunAgentInput.resume` 的省略/往返、`RunFinishedInterruptOutcome` 非空校验，以及 `RunFinishedEvent` 在旧版结构、成功结果、中断结果（含服务端生成的 JSON 结构）和显式 `null` outcome/result 下的往返。

### 示例
- Chatapp 将 `REASONING_*` 事件显示为临时的“💭 Reasoning…”气泡（新增 `MessageRole.REASONING` + `EphemeralType.REASONING`），与现有的工具调用/步骤临时模式一致。在 `RUN_FINISHED`、运行取消或运行错误时清除。处理 `REASONING_START` / `REASONING_END`、`REASONING_MESSAGE_START` / `REASONING_MESSAGE_CONTENT` / `REASONING_MESSAGE_END` 以及 `REASONING_MESSAGE_CHUNK`。
- 将所有 Kotlin 示例应用（chatapp、chatapp-java、chatapp-wearos、chatapp-swiftui、tools）从 `agui-core 0.3.0` 升级到 `0.4.1`，并从 Maven 消费已发布的构建产物，同时从四个 chatapp 变体的 settings 文件中移除 `includeBuild("../../library")` + dependencySubstitution 块。
- 将 chatapp 的 Kotlin 版本从 `2.1.20 → 2.2.20` 升级，以便 iOS 目标可以消费 `com.mikepenz:multiplatform-markdown-renderer-m3:0.37.0` klibs（要求 ABI 2.2.0+）。

## [0.4.0] - 2026-05-12

### 新增
- **推理事件（Reasoning events）**（[#1650](https://github.com/ag-ui-protocol/ag-ui/issues/1650)）。Kotlin SDK 现已支持在 TypeScript 和 Python SDK 中取代 `THINKING_*` 的七个 `REASONING_*` 事件。若无此变更，连接到发出 `REASONING_*` 事件的服务端的 Kotlin 客户端将因判别器 `type` 不匹配任何 `BaseEvent` 子类而多态反序列化失败。
  - `com.agui.core.types` 中新增 `EventType` 枚举项和 `BaseEvent` 子类：
    - `REASONING_START` / `ReasoningStartEvent(messageId)`
    - `REASONING_MESSAGE_START` / `ReasoningMessageStartEvent(messageId, role = "reasoning")`——`role` 在构造时校验，以匹配 TS 的 `z.literal("reasoning")` 和 Python 的 `Literal["reasoning"]` Schema。
    - `REASONING_MESSAGE_CONTENT` / `ReasoningMessageContentEvent(messageId, delta)`——要求 `delta` 非空。
    - `REASONING_MESSAGE_END` / `ReasoningMessageEndEvent(messageId)`
    - `REASONING_MESSAGE_CHUNK` / `ReasoningMessageChunkEvent(messageId?, delta?)`——两者均可选，以支持仅首个分块携带 `messageId` 的分块式流式传输。
    - `REASONING_END` / `ReasoningEndEvent(messageId)`
    - `REASONING_ENCRYPTED_VALUE` / `ReasoningEncryptedValueEvent(subtype, entityId, encryptedValue)`——`subtype` 校验为 `"tool-call"` 或 `"message"`（与 TS/Python 一致）。
- **`AgentState` 上的推理遥测。** 新增 `AgentState.reasoning: ReasoningTelemetryState?` 字段，按 `messageId` 跟踪一个或多个推理流。每个 `ReasoningStreamState` 携带该流累积的 `text`、`isActive` 标志以及附加的 `ReasoningEncryptedValue` 载荷。`AbstractAgent` 暴露对应的 `reasoning` 快照属性。
- **`DefaultApplyEvents` 接入 `REASONING_*` 事件。** 维护按 `messageId` 的流映射，使并发推理流可独立跟踪（这是之前的 `ThinkingTelemetryState` 无法表示的）。`REASONING_MESSAGE_CHUNK` 自动创建流，并在分块省略 `messageId` 时复用最后活跃的 `messageId`；`REASONING_ENCRYPTED_VALUE` 附加到最近活跃的流。推理映射和最后活跃 id 在 `RUN_STARTED` 时清除。
- **测试。**
  - `EventSerializationTest` 中新增 13 个序列化测试，覆盖全部七个新事件的往返、role/subtype 校验、空 delta 拒绝、可空分块字段、原始事件透传以及判别器路由。
  - `DefaultApplyEventsTest` 中新增 4 个应用处理器测试，覆盖单流生命周期、两个交错的并发流、加密值附加以及分块式自动填充。

### 已弃用
- 全部五个 `THINKING_*` 事件及对应的 `EventType` 枚举项均标记为 `@Deprecated(WARNING)`，并附 `ReplaceWith` 提示指向对应的 `REASONING_*` 类型。它们仍保留在传输协议中，并继续原样流经 `DefaultApplyEvents` 和 `EventVerifier`。计划在 1.0.0 中移除。
  - `ThinkingStartEvent` → `ReasoningStartEvent`
  - `ThinkingEndEvent` → `ReasoningEndEvent`
  - `ThinkingTextMessageStartEvent` → `ReasoningMessageStartEvent`
  - `ThinkingTextMessageContentEvent` → `ReasoningMessageContentEvent`
  - `ThinkingTextMessageEndEvent` → `ReasoningMessageEndEvent`
- `ThinkingTelemetryState` 和 `AgentState.thinking` / `AbstractAgent.thinking` 已弃用，改用 `ReasoningTelemetryState` 和 `reasoning`。旧字段仍由现有的 `THINKING_*` 应用处理器填充，因此下游消费者可继续正常工作。

### 集成方须知
- **传输兼容性不变。** 发出 `THINKING_*` 的服务端仍可像以前一样精确往返；发出 `REASONING_*` 的服务端现在可被解码。
- **`EventVerifier` 无变更。** 验证器原样透传 `REASONING_*` 事件，与 TypeScript SDK 一致。现有 `THINKING_*` 生命周期校验不变。
- **`Role` 枚举不变。** `ReasoningMessageStartEvent.role` 建模为受约束的 `String`，而非新增 `Role.REASONING` 值，以避免引入没有匹配 `Message` 子类的幽灵角色（并精确匹配 TS/Python 的字面量字符串 Schema）。

## [0.3.0] - 2026-05-09

### 新增
- `StatefulAgUiAgent` 将 `TOOL_CALL_START` / `TOOL_CALL_ARGS` / `TOOL_CALL_END` 和 `TOOL_CALL_RESULT` 事件记录到按线程的对话历史中。下一次运行的 `RunAgentInput.messages` 现在携带助手工具调用及匹配的工具结果，服务端（如 `ag_ui_adk`）需要这些来跨轮次配对挂起的工具调用记录。

### 变更
- 将 Ktor 从 3.1.3 升级到 3.2.4（与 Kotlin 2.1.x 兼容的最新版本）
- `ToolExecutionManager` 静默跳过不在本地注册表中的工具，而不再发出 `Error: Tool 'X' is not available` 的 `ToolMessage`。使服务端或中间件注入的工具规格（例如未在本地注册的 `@ag-ui/a2ui-middleware` 的 `render_a2ui` 工具）可透传而不污染历史。

### 修复
- 修复 chatapp-swiftui XCFramework 构建：移除已删除的 kotlin-a2ui 模块的过时复合构建替换，添加缺失的 a2ui-4k 目录条目，并在 iOS 兼容性上将 kotlinx-datetime 对齐到 0.7.1-0.6.x-compat 构建产物

### 示例
- 在 chatapp 示例中将 a2ui-4k 依赖从 0.8.0 更新到 0.8.1
- 从 chatapp-shared 移除不必要的核心库脱糖（minSdk 为 26）
- 将 chatapp 迁移到 `a2ui-4k 0.9.3` 以渲染 A2UI v0.9 界面。
- `ChatController` 在现有 v0.8 包装之外处理两种新的 `ActivitySnapshotEvent` 内容形态：(a) 单个 v0.9 信封（`content.version == "v0.9"`），以及 (b) 由 `@ag-ui/a2ui-middleware` Node 桥接发出的 `a2ui_operations` 数组。
- 通过新的 `RenderA2UiToolExecutor` 适配器（包装 `com.contextable.a2ui4k.agent.A2UiRenderTool`）将 `render_a2ui` 注册为客户端工具。该执行器直接驱动本地 `SurfaceStateManager` 并在本地完成 AG-UI 工具调用往返，因此中间件不再需要合成 `TOOL_CALL_RESULT`。
- 在下一条用户消息中内联发送 A2UI 操作结果（从聊天记录中过滤），以与不遵循 `forwardedProps` 的技术栈保持一致。

## [0.2.6] - 2026-01-14

### 变更
- 将最低 Android SDK 从 26 降至 24
- 将 Kotlin 版本从 2.2.20 降至 2.1.20
- 将 Android Gradle Plugin 从 8.10.1 更新到 8.12.0
- 修复构建产物 group ID 引用，从 `com.agui:` 改为 `com.ag-ui.community:`

## [0.2.5] - 2025-12-29

### 新增
- 代理订阅者系统，用于可选的生命周期和事件拦截。
- 分块转换和状态应用中的文本消息角色保真。

### 变更
- 默认应用流水线现在在变更状态前将每个事件路由到订阅者。
- 状态应用在构造流式消息时遵循 developer/system/user 角色。

### 测试
- 扩展了分块转换和状态应用的测试覆盖，涵盖角色传播和订阅者行为。

### 性能改进
- 使用 K2 编译器编译速度提升最高 2 倍
- 减少流式场景中的内存占用
- 因更好的优化而减小二进制体积
- 使用最新 kotlinx.coroutines 提升协程性能

## [0.1.0] - 2025-06-14

### 新增
- ag-ui-4k 客户端库首次发布
- Kotlin Multiplatform 的核心 AG-UI 协议实现
- 支持 SSE 的 HttpAgent 客户端，用于连接 AG-UI 代理
- 基于 Kotlin Flows 的事件驱动流式架构
- 使用密封类实现事件和消息的完整类型安全
- 支持 Android、iOS 和 JVM 平台
- 全面的事件类型（生命周期、消息、工具、状态）
- 带快照和增量的状态管理
- 用于人在回路工作流的工具集成
- 通过协程支持取消
- 使用 Kotlin 2.1.21 和 K2 编译器构建
- 由 Ktor 3.1.3 提供网络支持
- 使用 kotlinx.serialization 1.8.1 处理 JSON
- 全面的文档和示例
- GitHub Actions CI/CD 工作流
- Detekt 静态代码分析