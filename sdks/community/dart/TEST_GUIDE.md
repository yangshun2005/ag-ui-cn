# AG-UI Dart SDK 测试指南

## 运行测试

### 仅运行单元测试（推荐）
运行单元测试，排除需要外部服务的集成测试：

```bash
dart test --exclude-tags requires-server
```

### 运行所有测试
运行所有测试，包括集成测试（需要 TypeScript SDK 服务器环境）：

```bash
dart test
```

## 测试分类

### 单元测试（381+ 个测试）✅
- **SSE 组件**：解析器、客户端、消息、退避策略
- **类型**：基础类型、消息、工具、上下文
- **编码器/解码器**：客户端编解码、错误处理
- **事件**：事件类型、事件处理
- **客户端**：配置、错误处理

### 集成测试
这些测试需要 TypeScript SDK 的 Python 服务器正在运行：
- `simple_qa_test.dart` - 测试问答功能
- `tool_generative_ui_test.dart` - 测试基于工具的 UI 生成
- `simple_qa_docker_test.dart` - 基于 Docker 的集成测试

**注意**：集成测试标记为 `@Tags(['integration', 'requires-server'])`，使用 `--exclude-tags requires-server` 时默认跳过。

## 测试覆盖率

SDK 具有全面的单元测试覆盖，包括：
- 6 个 SSE 客户端基础测试
- 8 个 SSE 流解析测试
- 13 个 SSE 消息测试
- 67 个基础类型和 JSON 解码器测试
- 39 个错误处理测试
- 59 个事件类型测试
- 23 个客户端配置测试
- 以及更多...

## 已知限制

1. **SSE 重试测试**：有两个测试被跳过，因为 SSE 协议不支持 HTTP 错误时自动重试——这是协议限制，不是缺陷。

2. **集成测试**：需要 TypeScript SDK 基础设施，Dart SDK 目录结构中可能不具备。