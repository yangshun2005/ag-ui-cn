# @ag-ui/a2ui-toolkit

用于构建 A2UI 子代理工具的框架无关辅助函数。

各框架适配器（LangGraph、ADK、Mastra 等）将这些辅助函数与自身框架特定的粘合代码组合：工具装饰器、运行时访问器、模型绑定 + 调用。本包不依赖任何代理框架。

## 接口

- 常量：`A2UI_OPERATIONS_KEY`、`BASIC_CATALOG_ID`、`DEFAULT_SURFACE_ID`、
  `GENERATE_A2UI_TOOL_NAME`、`GENERATE_A2UI_TOOL_DESCRIPTION`、
  `GENERATE_A2UI_ARG_DESCRIPTIONS`
- 操作构建器：`createSurface`、`updateComponents`、`updateDataModel`
- `RENDER_A2UI_TOOL_DEF` — 内部结构化输出工具的 JSON Schema
- 状态 + 历史辅助函数：`buildContextPrompt`、`findPriorSurface`
- 提示词组合器：`buildSubagentPrompt`
- 高层编排：`prepareA2UIRequest`、`buildA2UIEnvelope`
- 输出包装器：`assembleOps`、`wrapAsOperationsEnvelope`、`wrapErrorEnvelope`

## 自定义工具名称与中间件关联

自定义生成包装器的 `toolName` 值仍可与中间件的默认嵌套行为配合使用。如果你选择启用 `a2uiGenerationToolNames` 来区分具有不同消息 ID 的兄弟调用，请包含所有包装器名称。示例和祖先限制参见[中间件配置指南](../../../../middlewares/a2ui-middleware/README.md#optional-sibling-call-disambiguation)。

## 另见

Python 对应版本位于
[`ag-ui-a2ui-toolkit`](../../../python/a2ui_toolkit)，以 snake_case 暴露相同的接口。