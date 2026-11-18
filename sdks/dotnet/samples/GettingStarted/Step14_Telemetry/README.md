# 第 14 步 — 遥测（OpenTelemetry 追踪）

本步骤展示如何用 **OpenTelemetry** 观测 AG-UI .NET 应用。协议本身不变——你只需组合标准中间件，并订阅两个活动源：

- `Experimental.AGUI.Server` — AG-UI **run** span（`agui.run`），每次运行一个，带 `agui.thread_id`、`agui.run_id`、`agui.parent_run_id`（恢复时）、`agui.run.outcome`（`success` / `interrupt` / `error` / `canceled`）和 `agui.events.count` 标签。
- `Experimental.AGUI.Client` — `AGUIChatClient` 在本地调用的客户端（前端）工具的 `execute_tool` span。
- `Experimental.Microsoft.Extensions.AI` — `UseOpenTelemetry()` 产生的 GenAI span（`chat`、`execute_tool`、`orchestrate_tools`），嵌套在 run span **之下**。

服务端还启用了 ASP.NET Core 插桩，客户端启用了 HttpClient 插桩，因此 W3C `traceparent` 头会随请求传递，整个交互是**一条 trace**。

## 运行

默认两个应用都把 span 打印到各自控制台（无外部依赖）。

```bash
# 终端 1 — 服务端（http://localhost:5014）
dotnet run --project Step14_Telemetry.Server

# 终端 2 — 客户端
dotnet run --project Step14_Telemetry.Client
```

客户端请求天气；服务端的假模型调用 `get_weather` 后端工具并作答。观察各控制台出现的 span。

## 将 trace 发送到无头接收端

在**两个**进程上设置 `OTEL_EXPORTER_OTLP_ENDPOINT`，即可从控制台导出器切换到 OTLP。任何 OTLP 接收端均可：

- **Aspire dashboard**（UI 在 `:18888`，OTLP 在 `:4317`）：

  ```bash
  npx -y @microsoft/aspire-cli dashboard run        # 或使用 mcr.microsoft.com/dotnet/aspire-dashboard 镜像
  $env:OTEL_EXPORTER_OTLP_ENDPOINT = "http://localhost:4317"   # 在两个终端中设置，然后按上述方式运行
  ```

- **OpenTelemetry Collector** 配合 `debug` 导出器（把收到的 span 打印到 collector 的 stdout）——将 `OTEL_EXPORTER_OTLP_ENDPOINT` 指向 collector 的 OTLP 端口。

- **`dotnet-counters`** 查看 GenAI meter 的实时指标：
  `dotnet-counters monitor -n Step14_Telemetry.Server --counters Experimental.Microsoft.Extensions.AI`。

## trace 的样子

单条 trace 横跨客户端和服务端：

```
agent conversation                         (client)
└─ chat                                     (client GenAI span — wraps the round trip)
   └─ POST http://localhost:5014/           (client HttpClient span)   ── traceparent ──▶
      └─ POST /                             (server ASP.NET request)
         └─ agui.run            {agui.thread_id, agui.run_id, agui.run.outcome=success, agui.events.count}
            └─ orchestrate_tools
               ├─ chat                      (gen_ai.operation.name=chat)        — model requests the tool
               ├─ execute_tool get_weather  (gen_ai.tool.name=get_weather)      — tool runs on the server
               └─ chat                      (gen_ai.operation.name=chat)        — model answers
```

对于**人在回路**流程，恢复是第二次运行：其 `agui.run` 带有指向第一次运行的 `agui.parent_run_id`，且（当客户端在同一活动下驱动两轮时）两次运行共享同一条 trace——因此中断及其恢复会一起呈现。