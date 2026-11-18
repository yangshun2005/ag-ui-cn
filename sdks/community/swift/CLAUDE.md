# AGUISwift

## 项目背景

AG-UI Swift SDK 是一个开源 Swift 框架，用于构建实现 Agent User Interaction Protocol（AG-UI）的 AI 智能体用户界面。它提供 Swift 应用与 AI 智能体之间的实时流式通信。

### 架构
```
AGUICore（基础层 - 协议类型 • 事件模型 • 消息定义）
    ↓
AGUIClient（底层客户端基础设施与传输实现 - HttpAgent • AbstractAgent • SseParser • EventStreamManager）
    ↓
    ├─→ AGUITools（用于扩展智能体能力的工具执行框架 - ToolExecutor • ToolRegistry • ToolExecutionManager）
    └─→ AGUIAgentSDK（常见智能体交互模式的高层 API - AgUiAgent • StatefulAgUiAgent • Builders）
         ↓
      应用

```

### 关键目录
- `Sources/AGUIAgentSDK/` - 常见智能体交互模式的高层 API。
- `Sources/AGUIClient/` - 底层客户端基础设施与传输实现。
- `Sources/AGUICore/` - 核心框架源码、协议类型、事件与消息定义。
- `Sources/AGUITools/` - 用于扩展智能体能力的工具执行框架。
- `Tests/` - 含模拟协议的测试套件

## 开发规范

### Swift 6.2 要求
- **最低要求**：iOS 15+、macOS 13+、Swift 5.9
- **并发**：默认使用 `async/await`、actor 和 `@MainActor` 隔离
- **数据安全**：所有公开类型必须符合 `Sendable`；使用 `@concurrent` 显式并行执行
- **值类型**：优先使用 `struct` 而非 `class`；共享可变状态使用 actor

### API 设计原则
- 协议优先：先定义行为契约，再实现
- 组合优于继承：使用协议组合（`Runnable & Observable`）
- 流式接口：适当使用 `@discardableResult` 支持链式调用
- 渐进式披露：简单默认值，高级配置通过构建器

### 命名约定
- 类型/协议：`UpperCamelCase`（如 `AgentProtocol`、`MemoryStore`）
- 方法：变更操作用动词短语（`execute`、`store`），访问器用名词短语（`configuration`）
- 布尔属性：使用 `is`、`has`、`should` 前缀
- 泛型：含义不清时使用描述性名称（`Element`、`Input`、`Output`）而非单字母

## 工作流

### 验证命令
```bash
swift build                           # 构建框架
swift test                            # 运行测试套件
swift test --filter AgentTests        # 运行指定测试
swift package plugin --allow-writing-to-package-directory swiftformat  # 格式化代码
```

### 提交前
1. 运行 `swift build` - 确保无编译错误
2. 运行 `swift test` - 所有测试必须通过
3. 运行 SwiftFormat - 代码必须已格式化（`swift package plugin --allow-writing-to-package-directory swiftformat`）
4. 运行 SwiftLint - 代码必须通过 lint 检查（`swiftlint lint` - 如需要可通过 Homebrew 安装）
5. 验证公开类型的 `Sendable` 符合性
6. 确保公开 API 有文档注释

### 分支命名
- `feature/agent-memory-system`
- `fix/orchestration-race-condition`
- `refactor/protocol-conformance`

## 技术指南

### 协议
- 使用 `associatedtype` 实现类型安全的泛型协议
- 提供带合理默认值的协议扩展
- 仅限类的协议用 `AnyObject` 约束标记
- 模式参考 `Sources/SwiftAgents/Core/Protocols/`

### 并发
- 优先使用结构化并发（`TaskGroup`、`async let`）而非非结构化并发
- UI 绑定代码用 `@MainActor` 标注
- 需要同步的共享状态使用 `actor`
- 跨隔离调用需要时显式应用 `nonisolated`

### 内存系统
- `ConversationMemory`：短期、受 token 限制的上下文
- `VectorMemory`：长期语义检索（需要嵌入提供方）
- `SummaryMemory`：压缩的对话历史

### 日志
- 使用 `swift-log` 实现跨平台兼容（Apple 平台 + Linux 服务器）
- 在应用启动时调用一次 `Log.bootstrap()` 配置日志
- 生产代码中禁止使用 `print()` 语句
- 使用分类日志器：
  - `Log.agents`：智能体生命周期与执行
  - `Log.memory`：内存系统操作
  - `Log.tracing`：可观测性与追踪事件
  - `Log.metrics`：性能与用量指标
  - `Log.orchestration`：多智能体协调
- 选择合适的日志级别：`.trace`、`.debug`、`.info`、`.notice`、`.warning`、`.error`、`.critical`
- **隐私说明**：与 `os.Logger` 不同，swift-log 不支持 `privacy:` 参数标注
  - 生产环境不得记录敏感用户数据、凭据或 PII
  - 配置日志处理器在运行时脱敏
  - 默认行为按原样记录所有插值
- 示例：`Log.memory.error("Failed to save: \(error.localizedDescription)")`
- 仅限 Apple 的代码可使用 `OSLogTracer`，其隐私标注包裹在 `#if canImport(os)` 中

### 测试策略（必须 TDD）
- **所有新功能和缺陷修复必须采用测试驱动开发**
- 所有协议在测试目标中都有对应的 `Mock*` 实现
- 使用 `XCTestExpectation` 或异步测试方法测试异步代码

### TDD 工作流
所有开发遵循红-绿-重构循环：

1. **红**：先写失败的测试
   - 通过测试用例定义预期行为
   - 使用 `test-specialist` 智能体设计全面的测试覆盖
   - 测试初始必须失败（验证测试有意义）

2. **绿**：编写最小实现
   - 只写足够让测试通过的代码
   - 不做过早优化或额外功能
   - 专注于满足测试断言

3. **重构**：提升代码质量
   - 在保持测试通过的前提下清理实现
   - 应用 Swift 模式与约定
   - 使用 `code-reviewer` 智能体验证重构

**TDD 准则**：
- 绝不在测试之前写实现
- 每个行为/功能一个测试用例
- 测试边界情况和错误条件
- 模拟外部依赖（LLM 提供方、网络）
- 开发过程中频繁运行 `swift test`


## 子智能体

使用专业智能体获得针对性专长。主动委派：

| 智能体 | 使用场景 |
|-------|-------------|
| `context-builder` | **长时间运行的任务**、收集需求、实现前调研 |
| `api-designer` | 设计公开 API、命名决策、流式接口 |
| `protocol-architect` | 类型层级、协议组合、POP 模式 |
| `concurrency-expert` | 异步代码、actor、Sendable 符合性、数据竞争安全 |
| `macro-engineer` | 实现或修改 Swift 宏 |
| `code-reviewer` | 代码变更后、提交前 |
| `test-specialist` | 编写测试、创建模拟、测试覆盖 |
| `framework-architect` | 多智能体编排模式、协调设计 |

### 上下文管理（长时间运行的任务）
**复杂或长时间任务开始时始终使用 `context-builder` 智能体**以：
- 实现前收集全面需求
- 调研现有代码库模式与约定
- 识别依赖与集成点
- 在委派给专业智能体前制定清晰路线图

**上下文保持工作流**：
1. **开始**：使用 `context-builder` 调研并收集上下文
2. **规划**：记录发现并制定实现计划
3. **执行**：带着清晰的上下文交接委派给专业智能体
4. **检查点**：多阶段任务重新调用 `context-builder`
5. **更新**：全程保持计划文档和记忆更新

**何时使用 `context-builder`**：
- 跨多个文件或组件的任务
- 需要理解现有架构的功能
- 需要根因调查的缺陷修复
- 任何预计超过几次交互的任务

### 委派准则
- **实现前**：使用 `context-builder` 调研，然后 `api-designer` 和 `protocol-architect`
- **实现中**：异步代码用 `concurrency-expert`，宏用 `macro-engineer`
- **变更后**：提交前始终运行 `code-reviewer`
- **测试**：委派给 `test-specialist` 处理模拟模式和覆盖（TDD：测试先行！）

**TDD + 子智能体工作流**（复杂功能）：
1. `context-builder` → 调研现有模式与需求
2. `test-specialist` → 先写失败测试（红阶段），保持异步/时间敏感测试确定性（无抖动）
3. `protocol-architect` → 设计满足测试的抽象
4. `api-designer` → 完善公开接口
5. `concurrency-expert` → 验证线程安全
6. `code-reviewer` → 最终审查（确保测试通过）

## 快速参考
- API 模式：见 `https://github.com/ag-ui-protocol/ag-ui/tree/main/sdks/community/kotlin`
- 协议设计：见 `https://github.com/ag-ui-protocol/ag-ui`
- 了解 AG-UI 如何补充并与 MCP 和 A2A 协作：见 `https://docs.ag-ui.com/llms-full.txt`

## Git 提交准则

创建提交时：
- 遵循约定式提交格式
- 首行保持在 50 字符以内
- 提交信息中不包含 AI 署名