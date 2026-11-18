# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中处理代码时提供指导。

如需为 AG-UI 应用添加 A2UI 渲染，或搭建 AG-UI + A2UI 快速上手项目，请使用 `skills/ag-ui-a2ui-integration/SKILL.md`。

## 常用开发命令

### TypeScript SDK（主要开发）
```bash

# 安装依赖（使用 pnpm）
pnpm install

# 构建所有包
pnpm build

# 运行开发模式
pnpm dev

# 运行代码检查
pnpm lint


# 运行类型检查
pnpm check-types

# 运行测试
pnpm test

# 格式化代码
pnpm format

# 清理构建产物
pnpm clean

# 完整清理后构建
pnpm build:clean
```

### Python SDK
```bash
# 进入 python-sdk 目录
cd python-sdk

# 安装依赖（使用 poetry）
poetry install

# 运行测试
python -m unittest discover tests

# 构建分发包
poetry build
```

### 运行特定集成测试
```bash
# 针对 TypeScript 包/集成
cd packages/<package-name>
pnpm test

# 运行单个测试文件
cd packages/<package-name>
pnpm test -- path/to/test.spec.ts
```

## 高层架构

AG-UI 是一个基于事件的协议，用于标准化智能体与用户的交互。代码库以 monorepo 形式组织，结构如下：

### 核心协议架构
- **事件驱动通信**：所有智能体与 UI 的通信都通过类型化事件完成（BaseEvent 及其子类型）
- **传输无关**：协议支持 SSE、WebSocket、HTTP 二进制和自定义传输
- **可观察模式**：使用 RxJS Observable 流式传输智能体响应

### 关键抽象
1. **AbstractAgent**：所有智能体必须实现的基类，提供 `run(input: RunAgentInput) -> Observable<BaseEvent>` 方法
2. **HttpAgent**：标准 HTTP 客户端，支持 SSE 和二进制协议，用于连接智能体端点
3. **事件类型**：生命周期事件（RUN_STARTED/FINISHED）、消息事件（TEXT_MESSAGE_*）、工具事件（TOOL_CALL_*）和状态管理事件（STATE_SNAPSHOT/DELTA）

### 仓库结构
- `/sdks/typescript/`：主要 TypeScript 实现
  - `/packages/`：核心协议包（@ag-ui/core、@ag-ui/client、@ag-ui/encoder、@ag-ui/proto）
- `/integrations/`：框架集成（langgraph、mastra、crew-ai 等）
- `/apps/`：示例应用，包括 AG-UI Dojo 演示查看器
- `/sdks/python/`：协议的 Python 实现
- `/docs/`：文档站点内容

### 集成模式
每个框架集成遵循相似的模式：
1. 实现 AbstractAgent 接口
2. 将框架特定事件转换为 AG-UI 协议事件
3. 同时提供 TypeScript 客户端和 Python 服务端实现
4. 包含演示 AG-UI 关键特性的示例（智能体对话、生成式 UI、人机协同等）

### 状态管理
- 使用 STATE_SNAPSHOT 表示完整状态
- 使用 STATE_DELTA 配合 JSON Patch（RFC 6902）实现高效的增量更新
- MESSAGES_SNAPSHOT 提供对话历史

### 多次顺序运行
- AG-UI 支持在单个事件流中进行多次顺序运行
- 每次运行必须完成（RUN_FINISHED）后才能开始新的运行（RUN_STARTED）
- 消息跨运行累积（例如 run1 的消息 + run2 的消息）
- 状态跨运行持续演进，除非通过 STATE_SNAPSHOT 显式重置
- 运行特定的跟踪信息（活跃消息、工具调用、步骤）在运行之间重置

### 开发工作流
- 使用 Nx 进行 monorepo 构建编排
- 每个包独立版本管理
- 集成测试验证协议合规性
- AG-UI Dojo 应用通过实时示例展示所有协议特性


<!-- nx configuration start-->
<!-- Leave the start & end comments to automatically receive updates. -->

# 使用 Nx 的通用准则

- 运行任务（例如 build、lint、test、e2e 等）时，始终优先通过 `nx` 运行（即 `nx run`、`nx run-many`、`nx affected`），而不是直接使用底层工具
- 你可以访问 Nx MCP 服务器及其工具，使用它们来帮助用户
- 回答关于仓库的问题时，优先使用 `nx_workspace` 工具了解工作区架构（如适用）
- 在单个项目中工作时，使用 `nx_project_details` MCP 工具分析和理解特定项目结构与依赖
- 关于 Nx 配置、最佳实践的问题，或不确定时，使用 `nx_docs` 工具获取相关的最新文档。始终使用它，而不是对 Nx 配置做假设
- 如果用户需要 Nx 配置或项目图错误的帮助，使用 `nx_workspace` 工具获取错误信息
- 关于 Nx 插件最佳实践，请查看 `node_modules/@nx/<plugin>/PLUGIN.md`。并非所有插件都有此文件——不可用时跳过即可。

<!-- nx configuration end-->