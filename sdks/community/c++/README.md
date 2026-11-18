# AG-UI C++ SDK

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![C++17](https://img.shields.io/badge/C++-17-blue.svg)](https://en.cppreference.com/w/cpp/17)
[![CMake](https://img.shields.io/badge/CMake-3.10+-064F8C.svg)](https://cmake.org/)

[AG-UI Protocol](https://github.com/ag-ui-protocol/ag-ui) 的社区 C++ 实现，提供用于 AI 代理与应用交互的 C++ SDK。该 SDK 实现了当前协议事件模型、流式传输、中间件、订阅者 Hook 和状态管理，覆盖本仓库测试所验证的功能。

## 特性

- **C++ 实现** - 跨平台支持，高性能
- **HTTP 连接** - 基于 libcurl，支持标准 HTTP 请求和流式 HTTP 请求
- **流处理** - SSE（Server-Sent Events）解析器，支持实时数据流
- **事件与状态管理** - 完整实现全部 27 种 AG-UI 事件类型及状态管理
- **中间件支持** - 灵活的请求/响应流水线，采用中间件架构
- **订阅者模式** - 支持外部订阅者处理事件
- **同步 API 设计** - 有意采用同步设计，最大化线程模型灵活性

## 架构与设计决策

### 同步 API 设计

`runAgent()` 方法使用**同步阻塞 API**。这是有意的设计决策，为不同应用架构提供最大灵活性。
不同应用有不同的线程需求。建议选择最适合自身架构的线程模型。

#### 实现

同步行为通过 libcurl 的阻塞 I/O 实现。事件在 SSE 流中到达时即被处理，回调在流处理过程中同步调用。此设计让你完全掌控线程，不会隐式创建线程，也不强制要求事件循环。

## 环境要求

### 构建依赖

- **CMake** (>= 3.10)
- **C++17 编译器** (GCC 7+、Clang 5+、MSVC 2017+)
- **nlohmann_json** (>= 3.2.0) - JSON 库
- **libcurl** - HTTP 客户端库

### 测试依赖（可选）

- **Google Test** (>= 1.10.0) - 仅在构建测试时需要

### 安装

#### macOS
```bash
# Install build dependencies
brew install cmake nlohmann-json curl

# Install Google Test (for testing)
brew install googletest
```

#### Ubuntu/Debian
```bash
# Install build dependencies
sudo apt-get update
sudo apt-get install cmake g++ pkg-config
sudo apt-get install nlohmann-json3-dev libcurl4-openssl-dev

# Install Google Test (for testing)
sudo apt-get install libgtest-dev
```

## 快速开始

### 构建 SDK

```bash
# Clone the repository
git clone https://github.com/ag-ui-protocol/ag-ui.git
cd ag-ui/sdks/community/c++

# Create build directory
mkdir build && cd build

# configure with tests enabled
cmake -DBUILD_TESTS=ON ..

# Build
make -j4
```

**注意**：指定 `BUILD_TESTS=ON` 时，CMake 会通过 `find_package(GTest REQUIRED)` 查找 Google Test。若未安装 Google Test，配置将报错失败。启用测试前请先安装 Google Test（参见[测试依赖](#测试依赖可选)）。

### 基本用法

```cpp
#include "agent/http_agent.h"

using namespace agui;

int main() {
    // Create an HTTP Agent
    auto agent = HttpAgent::builder()
        .withUrl("http://localhost:8080/api/agent/run")
        .withAgentId(AgentId("my-agent"))
        .build();
    
    // Create a subscriber to handle events
    class MySubscriber : public IAgentSubscriber {
        AgentStateMutation onTextMessageContent(
            const TextMessageContentEvent& event,
            const std::string& buffer,
            const AgentSubscriberParams& params) override {
            std::cout << event.delta;
            return AgentStateMutation();
        }
    };
    
    auto subscriber = std::make_shared<MySubscriber>();
    agent->subscribe(subscriber);
    
    // Run the agent
    RunAgentParams params;
    agent->runAgent(
        params,
        [](const RunAgentResult& result) {
            std::cout << "Success!" << std::endl;
        },
        [](const std::string& error) {
            std::cerr << "Error: " << error << std::endl;
        }
    );
    
    return 0;
}
```

## 测试

SDK 包含基于 **Google Test** 框架构建的完整测试套件，用于验证功能并演示用法。

### 测试前提条件

构建和运行测试前，请确保系统已安装 Google Test：

#### macOS
```bash
brew install googletest
```

#### Ubuntu/Debian
```bash
sudo apt-get install libgtest-dev
```

CMake 配置使用 `find_package(GTest REQUIRED)` 定位系统安装的 Google Test 库。若未找到 Google Test，CMake 将在配置阶段报错。

### 测试用例

**1. test_event_handler.cpp** - 事件处理器与订阅者行为测试
**2. test_state_manager.cpp** - 状态管理器与 JSON Patch 测试
**3. test_apply_module.cpp** - Apply 模块测试
**4. test_sse_parser.cpp** - SSE 解析器健壮性测试
**5. test_http_agent.cpp** - HttpAgent 运行生命周期与错误处理测试
**6. test_middleware.cpp** - 中间件系统测试
**7. test_sse_server.cpp** - Mock 服务器集成测试
**8. test_activity_events.cpp** - Activity 事件测试
**9. test_event_verifier.cpp** - 事件序列验证测试
**10. test_event_parser.cpp** - 全部 27 种事件类型的解析器往返测试
**11. test_agent_error.cpp** - AgentError 单元测试

### 运行测试

#### 1. 启动 Mock 服务器
**重要**：运行集成测试前务必先启动 Mock 服务器
```bash
cd tests/mock_server
python3 mock_ag_server.py --host 127.0.0.1 --port 8080
```

验证服务器运行状态：

```bash
# Health check
curl http://localhost:8080/health

# View available scenarios
curl http://localhost:8080/scenarios
```

#### 2. 使用 SSE 流测试 API

```bash
# Simple text scenario
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "simple_text"}'

# With thinking process
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "with_thinking"}'

# Custom delay (milliseconds)
curl -X POST http://localhost:8080/api/agent/run \
  -H "Content-Type: application/json" \
  -d '{"scenario": "simple_text", "delay_ms": 500}'
```

```bash
cd build

# Run individual tests
./tests/test_sse_parser
./tests/test_http_agent
./tests/test_middleware
./tests/test_sse_server
./tests/test_event_verifier
./tests/test_activity_events
./tests/test_event_handler
./tests/test_state_manager
./tests/test_apply_module
./tests/test_event_parser
./tests/test_agent_error
```


## 日志

C++ SDK 使用基于回调的日志系统。默认**禁用**日志，避免污染应用输出。

### 启用日志

```cpp
#include "core/logger.h"

// Set log callback
agui::Logger::setCallback([](agui::LogLevel level, const std::string& message) {
    const char* levelStr = "";
    switch (level) {
        case agui::LogLevel::Debug:   levelStr = "DEBUG"; break;
        case agui::LogLevel::Info:    levelStr = "INFO"; break;
        case agui::LogLevel::Warning: levelStr = "WARN"; break;
        case agui::LogLevel::Error:   levelStr = "ERROR"; break;
    }
    std::cout << "[AGUI][" << levelStr << "] " << message << std::endl;
});

// Optional: Set minimum log level (default is Info)
agui::Logger::setMinLevel(agui::LogLevel::Info);  // Only Info, Warning, Error
agui::Logger::setMinLevel(agui::LogLevel::Debug); // All messages
```

### 禁用日志

```cpp
agui::Logger::setCallback(nullptr);
```

### 集成到你的日志系统

```cpp
// Example: Integration with spdlog
agui::Logger::setCallback([](agui::LogLevel level, const std::string& message) {
    switch (level) {
        case agui::LogLevel::Debug:   spdlog::debug(message); break;
        case agui::LogLevel::Info:    spdlog::info(message); break;
        case agui::LogLevel::Warning: spdlog::warn(message); break;
        case agui::LogLevel::Error:   spdlog::error(message); break;
    }
});

// Example: Integration with custom logger
agui::Logger::setCallback([&myLogger](agui::LogLevel level, const std::string& message) {
    myLogger.log(static_cast<int>(level), message);
});
```

### 日志级别

- **Debug**：详细调试信息（线程 ID、请求体等）
- **Info**：一般信息（代理已创建、中间件已添加等）
- **Warning**：警告信息（SSE 解析器错误等）
- **Error**：错误信息（HTTP 失败、校验错误等）

## 项目结构

```
c++/
├── src/
│   ├── agent/          # Agent implementations
│   ├── core/           # Core types and utilities
│   ├── http/           # HTTP service layer
│   ├── middleware/     # Middleware system
│   ├── stream/         # SSE parser
│   └── apply/          # State application
├── tests/
│   ├── mock_server/    # Mock AG-UI server
│   ├── test_*.cpp      # Test suites
│   └── *.md            # Test documentation
├── CMakeLists.txt      # Build configuration
└── README.md           # This file
```

## 贡献

欢迎为 AG-UI C++ SDK 做贡献！你可以这样参与：

1. **Fork 仓库**
2. **创建功能分支**（`git checkout -b feature/amazing-feature`）
3. **为新功能添加测试**
4. **确保所有测试通过**（`ctest -V`）
5. **提交更改**（`git commit -m 'Add amazing feature'`）
6. **推送到分支**（`git push origin feature/amazing-feature`）
7. **发起 Pull Request**

## 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。

## 联系方式

- **Issues**：[GitHub Issues](https://github.com/ag-ui-protocol/ag-ui/issues)

---

**注意**：本项目由社区驱动。感谢你的反馈与贡献！