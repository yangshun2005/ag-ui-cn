<!-- nx configuration start-->
<!-- 保留开始和结束注释，以便自动接收更新。 -->

# 使用 Nx 的通用指南

- 如需为 AG-UI 应用添加 A2UI 渲染，或搭建 AG-UI + A2UI 快速入门脚手架，请使用
  `skills/ag-ui-a2ui-integration/SKILL.md`。
- 运行任务（例如 build、lint、test、e2e 等）时，始终优先通过 `nx` 运行（即 `nx run`、`nx run-many`、`nx affected`），不要直接使用底层工具
- 你可以访问 Nx MCP 服务器及其工具，请用它们帮助用户
- 回答关于仓库的问题时，先使用 `nx_workspace` 工具了解工作区架构（如适用）。
- 在单个项目中工作时，使用 `nx_project_details` MCP 工具分析并理解具体项目结构和依赖
- 关于 Nx 配置、最佳实践的问题，或不确定时，使用 `nx_docs` 工具获取相关的最新文档。始终使用它，不要对 Nx 配置做假设
- 如果用户遇到 Nx 配置或项目图错误，使用 `nx_workspace` 工具获取错误信息
- 关于 Nx 插件最佳实践，请查看 `node_modules/@nx/<plugin>/PLUGIN.md`。并非所有插件都有此文件——不可用时跳过即可。

<!-- nx configuration end-->