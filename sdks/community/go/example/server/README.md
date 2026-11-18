# AG-UI Go 示例服务器

本示例是一个 Fiber v3 AG-UI 服务器，用于本地开发和 Dojo 测试。
它基于 Go Dojo 服务器示例，用更广的路由面替换了旧的极简 `/agentic` 示例。

## 运行

需要 Go 1.25 或更高版本。Fiber v3.3.0 要求 Go 1.25，因此 `go mod tidy`
会将本示例模块更新为 `go 1.25.0`。

```bash
cd sdks/community/go/example/server
OPENAI_API_KEY=... go run ./cmd
```

默认服务器地址为 `http://127.0.0.1:8080`。

服务器使用 `github.com/cloudwego/eino-ext/components/model/openai` 中可复现的
`MODEL_PROVIDER=openai` 路径。独立源码中仅用于原型的 provider 和工具依赖已从本
monorepo 示例中移除。只读 `file_read` 工具在 `internal/agent/tools.go` 中本地实现。

## 环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `HOST` | `127.0.0.1` | 服务器绑定主机。 |
| `PORT` | `8080` | 服务器端口。 |
| `MODEL_PROVIDER` | `openai` | 本 monorepo 示例仅支持 `openai`。 |
| `MODEL` | `gpt-4o` | 传给 OpenAI Eino provider 的模型。 |
| `OPENAI_API_KEY` | 未设置 | 模型路由和多模态辅助客户端必需。 |
| `AGENT_WORKSPACE` | 进程工作目录 | `file_read` 工具的只读根目录。请谨慎设置。 |
| `AGENT_AUTO_APPROVE` | `false` | 绕过 `/agentic` 的审批中断路径。 |
| `AGENT_MAX_ITERATIONS` | `8` | 模型/工具循环迭代预算。 |
| `CORS_ENABLED` | `true` | 为本地 UI 开发启用宽松 CORS。 |
| `AGENTIC_UI_PACE_MS` | `600` | `/agentic_generative_ui` 的步骤延迟。 |

## 路由

| 路由 | 说明 |
| --- | --- |
| `GET /` | 健康/配置元数据和路由列表。 |
| `POST /agentic` | 主只读 agent 循环。 |
| `POST /agentic_chat` | Dojo 前端工具聊天路由。 |
| `POST /backend_tool_rendering` | Dojo 路由别名，使用前端工具聊天姿态。 |
| `POST /human_in_the_loop` | 审批路由。`X-AG-Approval: off` 或 `?approval=off` 禁用审批门。 |
| `POST /agentic_generative_ui` | 确定性多步生成式 UI 路由。 |
| `POST /tool_based_generative_ui` | 客户端工具生成式 UI 路由。 |
| `POST /shared_state` | 食谱共享状态路由。 |
| `POST /predictive_state_updates` | 文档预测状态更新路由。 |
| `POST /agentic_chat_multimodal` | 使用 OpenAI provider 路径的多模态聊天路由。 |
| `POST /image-gen` | OpenAI 图像生成辅助。 |
| `POST /vision` | OpenAI 视觉辅助。 |
| `POST /audio` | OpenAI 音频转录辅助。 |
| `POST /document` | OpenAI 文档分析辅助。 |

所有 `POST` 路由接受 AG-UI `RunAgentInput` JSON，除非特定处理器另有说明。
SSE 响应使用 AG-UI 事件流格式。

## Dojo

单独运行本服务器，并配置 Dojo 将相关集成运行时 URL 指向 `http://127.0.0.1:8080`。
本服务器位于 Go SDK 示例下，而非 `integrations/.../examples` 包，因此本次变更
不会自动将其注册为一等 Dojo 集成。

## Go 客户端

Go 示例客户端默认端点与本服务器对齐：

```text
http://localhost:8080/agentic
```

如果在不同主机或端口运行服务器，请向客户端传入显式端点。

## 验证

```bash
cd sdks/community/go/example/server
gofmt -w ./cmd ./internal
go mod tidy
go test ./...
go test ./... -run 'Test.*Route|Test.*App|Test.*SSE'
rg -n 'github.com/mattsp1290/ag-ui-go-server-example|/Users/punk1290/git|v0.0.0-00010101000000-000000000000' go.mod go.sum cmd internal
```

需要凭据的模型调用需要 `OPENAI_API_KEY`；需要它的集成测试在该变量未设置时跳过。