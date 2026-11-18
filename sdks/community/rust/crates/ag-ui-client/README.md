# AG-UI Rust 客户端

用于对接 AG-UI 协议的 Rust 客户端。客户端 API 的设计尽可能贴近 TypeScript 客户端。但有一个关键区别：状态和消息目前还不是 [`Agent`](src/agent.rs) 实现的属性，因为那会要求简单实现使用 `&mut self`。此项工作仍在进行中。

## 示例

运行每个示例前，请务必阅读启动对应 AG-UI 服务器的说明。

### 基础示例

```rust,no_run
// no_run: 此示例连接真实的 AG-UI 后端（127.0.0.1:3001），
// 该后端在 `cargo test` 期间不可用。但仍会编译/类型检查。
use std::error::Error;
use ag_ui_client::{core::types::Message, Agent, HttpAgent, RunAgentParams};

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>>{
	let agent = HttpAgent::builder()
		.with_url_str("http://127.0.0.1:3001/")?
		.build()?;

	let message = Message::new_user("Can you give me the current temperature in New York?");
	// 创建运行参数
	let params = RunAgentParams::new().add_message(message);

	// 使用订阅者运行 agent
	let result = agent.run_agent(&params, ()).await?;

    println!("{:#?}", result);
    Ok(())
}
```

更多示例请查看 [examples 文件夹](examples)。