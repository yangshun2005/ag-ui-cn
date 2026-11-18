# ag-ui-a2ui-toolkit

用于构建 A2UI 子代理工具的框架无关辅助函数。

各框架适配器（LangGraph、ADK、Mastra 等）将这些辅助函数与各自的框架特定胶水代码组合使用：工具装饰器、运行时访问器、模型绑定 + 调用。本包不依赖任何代理框架。

## 接口

- 常量：`A2UI_OPERATIONS_KEY`、`BASIC_CATALOG_ID`、`DEFAULT_SURFACE_ID`、
  `GENERATE_A2UI_TOOL_NAME`、`GENERATE_A2UI_TOOL_DESCRIPTION`、
  `GENERATE_A2UI_ARG_DESCRIPTIONS`
- 操作构建器：`create_surface`、`update_components`、`update_data_model`
- `RENDER_A2UI_TOOL_DEF` — 内部结构化输出工具的 JSON Schema
- 状态 + 历史辅助函数：`build_context_prompt`、`find_prior_surface`
- 提示词组合器：`build_subagent_prompt`
- 高层编排：`prepare_a2ui_request`、`build_a2ui_envelope`
- 输出包装器：`assemble_ops`、`wrap_as_operations_envelope`、
  `wrap_error_envelope`

## 另见

TypeScript 对应版本位于
[`@ag-ui/a2ui-toolkit`](../../typescript/packages/a2ui-toolkit)，以 camelCase 暴露相同接口。