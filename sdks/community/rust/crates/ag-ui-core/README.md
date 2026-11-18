# AG-UI 核心类型

本仓库包含使用 AG-UI 协议所需的 Rust 类型。基于 `serde` 实现，支持（反）序列化。

包含以下内容：

* [消息类型](src/types/message.rs)
* [事件类型](src/event.rs)
* [状态 trait 约束](src/state.rs)
* [输入类型](src/types/input.rs)
* [工具类型](src/types/tool.rs)
* [上下文类型](src/types/context.rs)
* [ID（new）类型](src/types/ids.rs)

设计上配合 [`ag-ui-client`](../ag-ui-client) 使用。