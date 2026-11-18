# AG-UI Mock 服务器

用于测试 C++ SDK 的本地 Mock 服务器，支持 HTTP 流式数据传输和全部 23 种 AG-UI 协议事件类型。

## 功能特性

**完整的 AG-UI 协议支持**
- 100% 覆盖 23 种事件类型
- SSE（Server-Sent Events）流式响应
- 状态管理（快照和增量更新）
- 工具调用模拟
- 思考过程模拟

**预定义测试场景**
- `simple_text`：简单文本消息
- `with_thinking`：带思考过程
- `with_tool_call`：带工具调用
- `with_state`：带状态管理
- `error`：错误场景
- `all_events`：全部事件类型演示

**易于使用**
- 零依赖（仅需 Python 3.6+）
- 命令行启动
- RESTful API
- 支持 CORS

## 快速开始

### 1. 启动服务器

```bash
# 使用默认端口 8080
python3 tests/mock_server/mock_ag_server.py

# 指定端口
python3 tests/mock_server/mock_ag_server.py --port 9090

# 指定主机和端口
python3 tests/mock_server/mock_ag_server.py --host 127.0.0.1 --port 8080
```

### 2. 验证服务器运行

```bash
# 健康检查
curl http://localhost:8080/health

# 查看可用场景
curl http://localhost:8080/scenarios
```

### 3. 测试 Agent API

```bash
# 简单文本场景
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "simple_text"}'

# 带思考过程
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "with_thinking"}'

# 自定义延迟（毫秒）
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "simple_text", "delay_ms": 500}'
```

## API 文档

### GET /health

健康检查端点

**响应示例：**
```json
{
  "status": "ok",
  "server": "AG-UI Mock Server",
  "version": "1.0.0"
}
```

### GET /scenarios

获取可用测试场景列表

**响应示例：**
```json
{
  "scenarios": [
    "simple_text",
    "with_thinking",
    "with_tool_call",
    "with_state",
    "error",
    "all_events"
  ],
  "description": {
    "simple_text": "简单文本消息",
    "with_thinking": "带思考过程",
    "with_tool_call": "带工具调用",
    "with_state": "带状态管理",
    "error": "错误场景",
    "all_events": "全部事件类型"
  }
}
```

### POST /api/agent/run

运行 Agent 并返回 SSE 流式响应

**请求参数：**
```json
{
  "scenario": "simple_text",  // 场景名称（可选，默认：simple_text）
  "delay_ms": 100             // 事件间延迟，单位毫秒（可选，默认：100）
}
```

**响应格式：** SSE（Server-Sent Events）

**响应示例：**
```
data: {"type":"RUN_STARTED","runId":"run_001"}

data: {"type":"TEXT_MESSAGE_START","messageId":"msg_001","role":"assistant"}

data: {"type":"TEXT_MESSAGE_CONTENT","messageId":"msg_001","delta":"Hello, "}

data: {"type":"TEXT_MESSAGE_CONTENT","messageId":"msg_001","delta":"world!"}

data: {"type":"TEXT_MESSAGE_END","messageId":"msg_001"}

data: {"type":"RUN_FINISHED","runId":"run_001"}

```

## 测试场景说明

### 1. simple_text - 简单文本消息

最基础的文本消息流程。

**事件序列：**
1. RUN_STARTED
2. TEXT_MESSAGE_START
3. TEXT_MESSAGE_CONTENT（多次）
4. TEXT_MESSAGE_END
5. RUN_FINISHED

**适用场景：**
- 基础功能测试
- 快速连接验证
- 性能基准测试

### 2. with_thinking - 带思考过程

模拟 AI 思考过程。

**事件序列：**
1. RUN_STARTED
2. THINKING_START
3. THINKING_TEXT_MESSAGE_START
4. THINKING_TEXT_MESSAGE_CONTENT
5. THINKING_TEXT_MESSAGE_END
6. THINKING_END
7. TEXT_MESSAGE_START
8. TEXT_MESSAGE_CONTENT
9. TEXT_MESSAGE_END
10. RUN_FINISHED

**适用场景：**
- 测试思考事件处理
- 验证事件过滤
- UI 展示测试

### 3. with_tool_call - 带工具调用

模拟工具调用流程。

**事件序列：**
1. RUN_STARTED
2. TEXT_MESSAGE_START/CONTENT/END
3. TOOL_CALL_START
4. TOOL_CALL_ARGS（多次）
5. TOOL_CALL_END
6. TOOL_CALL_RESULT
7. TEXT_MESSAGE_START/CONTENT/END
8. RUN_FINISHED

**适用场景：**
- 测试工具调用处理
- 验证参数拼接
- 工具结果处理

### 4. with_state - 带状态管理

模拟状态更新流程。

**事件序列：**
1. RUN_STARTED
2. STATE_SNAPSHOT
3. TEXT_MESSAGE_START/CONTENT/END
4. STATE_DELTA
5. TEXT_MESSAGE_START/CONTENT/END
6. STATE_DELTA
7. RUN_FINISHED

**适用场景：**
- 测试状态管理
- 验证增量更新
- 状态同步测试

### 5. error - 错误场景

模拟错误情况。

**事件序列：**
1. RUN_STARTED
2. TEXT_MESSAGE_START/CONTENT
3. RUN_ERROR

**适用场景：**
- 错误处理测试
- 异常恢复验证
- 错误日志测试

### 6. all_events - 全部事件类型

完整演示全部 23 种事件类型。

**适用场景：**
- 完整功能测试
- 协议兼容性验证
- 集成测试

## C++ SDK 集成示例

### 基本用法

```cpp
#include "agent/http_agent.h"

using namespace agui;

int main() {
    // 创建 Agent
    auto agent = HttpAgent::builder()
        .withUrl("http://localhost:8080")
        .withAgentId(AgentId("test_agent"))
        .build();
    
    // 创建订阅者
    class MySubscriber : public IAgentSubscriber {
        AgentStateMutation onTextMessageContent(
            const TextMessageContentEvent& event) override {
            std::cout << event.delta;
            return AgentStateMutation();
        }
    };
    
    auto subscriber = std::make_shared<MySubscriber>();
    agent->subscribe(subscriber);
    
    // 运行 Agent
    RunAgentParams params;
    params.input.message = "Hello";
    params.input.scenario = "simple_text";  // 指定场景
    
    agent->runAgent(
        params,
        [](const RunAgentResult& result) {
            std::cout << "\nSuccess!" << std::endl;
        },
        [](const AgentError& error) {
            std::cerr << "Error: " << error.message << std::endl;
        }
    );
    
    return 0;
}
```

### 测试不同场景

```cpp
// 测试思考过程
params.input.scenario = "with_thinking";
agent->runAgent(params, onSuccess, onError);

// 测试工具调用
params.input.scenario = "with_tool_call";
agent->runAgent(params, onSuccess, onError);

// 测试状态管理
params.input.scenario = "with_state";
agent->runAgent(params, onSuccess, onError);

// 测试错误处理
params.input.scenario = "error";
agent->runAgent(params, onSuccess, onError);
```

### 自定义延迟

```cpp
// 快速测试（50ms 延迟）
params.input.delay_ms = 50;

// 慢速测试（500ms 延迟）
params.input.delay_ms = 500;

// 无延迟（压力测试）
params.input.delay_ms = 0;
```

## 自动化测试集成

### 在测试脚本中使用

```bash
#!/bin/bash

# 启动 Mock 服务器
python3 tests/mock_server/mock_ag_server.py --port 8080 &
SERVER_PID=$!

# 等待服务器启动
sleep 2

# 运行测试
./build/test_http_agent
./build/test_integration

# 停止服务器
kill $SERVER_PID
```

### Docker 集成

```dockerfile
# Dockerfile
FROM python:3.9-slim

WORKDIR /app
COPY tests/mock_server/mock_ag_server.py .

EXPOSE 8080

CMD ["python3", "mock_ag_server.py", "--host", "0.0.0.0", "--port", "8080"]
```

```bash
# 构建镜像
docker build -t ag-ui-mock-server .

# 运行容器
docker run -d -p 8080:8080 ag-ui-mock-server

# 停止容器
docker stop <container_id>
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  mock-server:
    build: .
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 5s
      timeout: 3s
      retries: 3
```

```bash
# 启动
docker-compose up -d

# 停止
docker-compose down
```

## CI/CD 集成

### GitHub Actions

```yaml
name: C++ SDK Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'
      
      - name: Start Mock Server
        run: |
          python3 tests/mock_server/mock_ag_server.py &
          sleep 2
      
      - name: Build and Test
        run: |
          mkdir build && cd build
          cmake -DBUILD_TESTS=ON ..
          make
          ./test_http_agent
          ./test_integration
```

## 故障排查

### 问题 1：端口已被占用

**错误信息：**
```
OSError: [Errno 48] Address already in use
```

**解决方案：**
```bash
# 查找占用端口的进程
lsof -i :8080

# 终止进程
kill -9 <PID>

# 或使用其他端口
python3 mock_ag_server.py --port 9090
```

### 问题 2：连接被拒绝

**错误信息：**
```
Connection refused
```

**解决方案：**
1. 确认服务器已启动
2. 检查防火墙设置
3. 验证端口号是否正确
4. 使用 `curl` 测试连接

### 问题 3：SSE 流中断

**可能原因：**
- 网络超时
- 客户端断开连接
- 服务器崩溃

**解决方案：**
1. 增加超时时长
2. 添加重连逻辑
3. 检查服务器日志

## 性能测试

### 基准测试

```bash
# 使用 Apache Bench
ab -n 1000 -c 10 -p request.json -T application/json \
  http://localhost:8080/api/agent/run

# 使用 wrk
wrk -t4 -c100 -d30s --latency \
  -s post.lua http://localhost:8080/api/agent/run
```

### 压力测试

```bash
# 无延迟高并发
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "simple_text", "delay_ms": 0}'
```

## 扩展开发

### 添加自定义场景

编辑 `mock_ag_server.py`，在 `SCENARIOS` 字典中添加：

```python
SCENARIOS = {
    # ... 现有场景 ...
    
    "my_custom_scenario": [
        AGUIEvent.run_started("run_custom"),
        AGUIEvent.text_message_start("msg_custom", "assistant"),
        AGUIEvent.text_message_content("msg_custom", "Custom content"),
        AGUIEvent.text_message_end("msg_custom"),
        AGUIEvent.run_finished("run_custom")
    ]
}
```

### 添加新事件类型

在 `AGUIEvent` 类中添加静态方法：

```python
@staticmethod
def my_custom_event(param1, param2):
    return {
        "type": "MY_CUSTOM_EVENT",
        "param1": param1,
        "param2": param2
    }
```

## 最佳实践

1. **开发期间使用 Mock 服务器**
   - 快速迭代
   - 无需真实服务
   - 可控的测试环境

2. **集成测试使用真实服务**
   - 验证协议兼容性
   - 端到端测试
   - 生产环境模拟

3. **性能测试使用无延迟模式**
   - `delay_ms: 0`
   - 压力测试
   - 性能基准测试

4. **错误测试使用错误场景**
   - 异常处理
   - 错误恢复
   - 日志验证

## 总结

AG-UI Mock 服务器提供：

**完整的协议支持** - 23 种事件类型
**灵活的测试场景** - 6 个预定义场景
**易于集成** - 零依赖，命令行启动
**生产级特性** - SSE 流式响应，支持 CORS
**开发者友好** - 详细文档，示例代码

使用 Mock 服务器可以：
- 加速开发迭代
- 提高测试覆盖率
- 降低测试成本
- 提升代码质量

**开始使用：**
```bash
python3 tests/mock_server/mock_ag_server.py
```

**获取帮助：**
```bash
python3 tests/mock_server/mock_ag_server.py --help
```