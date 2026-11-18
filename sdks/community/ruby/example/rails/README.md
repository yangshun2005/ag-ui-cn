# AG-UI Ruby 示例：Rails

本示例演示如何在 **Rails (Puma)** 中使用 **Server-Sent Events (SSE)** 暴露一个最小的 AG-UI 兼容端点。

`POST /` 端点会流式发送以下事件（顺序与 Python 示例相同）：

- `RunStartedEvent`
- `TextMessageStartEvent`
- `TextMessageContentEvent`
- `TextMessageEndEvent`
- `RunFinishedEvent`

## 前置条件

- **Ruby**：3.0 或更高版本

```bash
# 检查 Ruby 版本
ruby --version
```

- **Bundle**：2.5.14 或更高版本

```bash
# 检查 Bundle 版本
bundle --version
```

## 安装

### 1. 克隆仓库

```bash
# 克隆 AG-UI 仓库
git clone https://github.com/ag-ui-protocol/ag-ui.git
cd ag-ui
```

### 2. 安装依赖

```bash
# 进入 Rails 示例目录
cd sdks/community/ruby/example/rails
# 安装依赖
bundle install
```

## 运行示例

1. 启动 Rails 服务器（Puma）

    ```bash
    bundle exec puma -C config/puma.rb config.ru
    ```

2. 在另一个终端中用 curl 测试端点

    ```bash
    curl -N \
      -H 'Accept: text/event-stream' \
      -H 'Content-Type: application/json' \
      -d '{"thread_id":"thread_123","run_id":"run_123"}' \
      http://localhost:3000/
    ```

空请求体也可以（服务器会生成 `thread_id`/`run_id`）：

```bash
curl -N -H 'Accept: text/event-stream' -H 'Content-Type: application/json' -d '{}' http://localhost:3000/
```

## 使用 ActionController::Live 的替代方案（概念概述）

你也可以直接使用 `ActionController::Live` 来流式发送事件，而不使用 `with_stream`。本示例为简单起见并保持与 Rails 7.1 一致，使用了 `with_stream`。

本节介绍如何直接使用 `ActionController::Live` 实现同样的 SSE 兼容 AG-UI 端点。核心思路：

1. **引入模块**  
   在控制器中引入 `ActionController::Live` 以启用流式响应。

2. **设置 SSE 响应头**  
   为 Server-Sent Events 流配置响应头：
   - `Content-Type: text/event-stream`
   - `Cache-Control: no-cache`
   - `Connection: keep-alive`

3. **使用 `response.stream`**  
   在 `begin`/`ensure` 块中向 `response.stream` 写入编码后的 AG-UI 事件，确保：
   - 使用 `AgUiProtocol::Encoder::EventEncoder` 编码每个事件
   - 如有需要，每次写入后刷新（`response.stream.write(...)`）
   - 始终在 `ensure` 中通过 `response.stream.close` 关闭流

4. **处理异常和客户端断开**  
   需处理以下情况：
   - 客户端断开时的 `IOError`
   - 用于日志记录 / 清理的一般异常
   - 确保出错后不再写入

5. **事件生命周期**  
   按与其他示例相同的顺序发送事件（如 `RunStarted`、文本内容事件、`RunFinished`）。从概念上讲，控制器动作会：
   - 解析请求体（thread/run ID 等）
   - 创建相应的 AG-UI 事件
   - 逐个以 SSE 形式流式发送给客户端

这种方式让你能细粒度控制流式行为，同时完全兼容 AG-UI 协议客户端。