# 跨语言 Fixtures

本目录中的 Fixtures 由多个 AG-UI SDK 共享。它们的存在是为了让线格式预期只写**一次**，每个 SDK 都遵循同一份文本，而不是各自携带可能漂移的副本。

这里的 fixture 是纯 JSON，不含任何语言特定假设。仅服务于单一实现的 SDK 专属 fixture 留在该 SDK 的测试树内。

## `null-omission.json`

约定：生产者在字段无值时将其**省略**，而不是写成 `null`。

TypeScript 天然如此（`JSON.stringify` 会丢弃 `undefined`）。Python 和 .NET 不是——它们的序列化器默认写 `null`——这一分歧在源头修复前，已让协议付出了三个接收侧容错补丁的代价（`TOOL_CALL_START.parentMessageId`、`TOOL_CALL_CHUNK.parentMessageId`、`RUN_FINISHED.outcome` 至今仍为兼容旧生产者而接受 `null`）。该 fixture 固定此行为，以免需要第四个补丁。

### 结构

```jsonc
{
  "stream": [
    {
      "name": "run_finished_without_result_or_outcome",
      "producedBy": ["typescript", "python", "dotnet"],
      "note": "why this case is here",
      "input": {
        "type": "RUN_FINISHED",
        "threadId": "thread_1",
        "runId": "run_1",
      },
      "expected": {
        "type": "RUN_FINISHED",
        "threadId": "thread_1",
        "runId": "run_1",
      },
    },
  ],
}
```

每个 SDK 的测试遍历 `stream`，对每个在 `producedBy` 中列出自身名称的用例：

1. 将 `input` 反序列化为 SDK 的原生事件类型，
2. 通过 SDK 的官方生产者路径（其事件编码器 / SSE 格式化器）重新序列化，
3. 断言输出的 JSON 解析后恰好等于 `expected`。

由于 `expected` 是精确匹配，多余的 `null` 会导致用例失败；而约定**确实**携带的 `null`（单个元数据值、状态快照内的值或 JSON Patch 操作内的值）必须仍然存在。用例应自上而下作为一个合理的连续事件流来阅读。

`producedBy` 仅用于 SDK 确实未实现的事件类型。它不是 SDK 失败用例的逃生通道。此处每个用例目前都列出全部三个 SDK——chunk 事件是最后的豁免项，而 .NET 此后已实现它们；如果你发现自己想用更短的列表，请在 `note` 中准确说明缺失的 SDK 缺少什么，以便该 SDK 具备后删除该条目。

### 消费者

| SDK        | 测试                                                                       |
| ---------- | -------------------------------------------------------------------------- |
| TypeScript | `sdks/typescript/packages/encoder/src/__tests__/null-omission.test.ts`     |
| Python     | `sdks/python/tests/test_null_omission.py`                                  |
| .NET       | `sdks/dotnet/tests/AGUI.Abstractions.UnitTests/NullOmissionFixtureTest.cs` |

这些 fixture 测试检查各 SDK 之间是否一致。每个 SDK 另有反射驱动的测试，遍历其定义的**每一个**线类型——不仅是此处列出的——并在出现约定不允许的 `null` 时失败。在此添加用例并不能取代那次更广泛的扫描；两者捕获的问题不同。

### 接入新消费者：让构建系统看到本目录

此处的 fixture 位于每个 SDK 的项目目录之外，因此仅查看项目内部来决定重跑内容的构建系统不会察觉其变化。每个消费者都需要显式处理，否则此处的编辑可能留下过期的绿色结果：

- **.NET** 将该文件作为 `EmbeddedResource` 链接（见 `AGUI.Abstractions.UnitTests.csproj`）。MSBuild 会跟踪 `EmbeddedResource` 项，因此已处理。
- **Python** 由 `unittest` 直接运行，无缓存层。无需处理。
- **TypeScript** 在 Nx 下运行，Nx 基于 `{projectRoot}/**/*` 缓存 `test`——本目录不在其中。因此 `@ag-ui/encoder` 在其 `package.json` 中设置 `nx.targets.test.cache: false`；其测试套件耗时远低于一秒，始终运行比承担风险更划算。

新消费者还需要第二件事：**CI 触发器**。本目录位于 `.github/workflows/unit-*.yml` 中所有路径列表之外，因此在将 `sdks/fixtures/**` 加入每个工作流之前，仅编辑此处 fixture 的 PR 不会运行任何 SDK **测试**任务。并非完全没有任务：`typecheck-typescript.yml` 通过其笼统的 `sdks/**` 条目匹配到它并编译了工作区，而编译不会因 fixture 内容而失败。没有任何执行这些文档的任务运行。现在三个单元测试工作流都在其 `push` 和 `pull_request` 过滤器中列出了 `sdks/fixtures/**`；第四个工作流中的新消费者需要同样的行。

显而易见的替代方案——在 `nx.json` 中向 `test` 目标的 `inputs` 添加 `{workspaceRoot}/sdks/fixtures/**/*`——**不可行**，且已尝试过。在此工作区的 Nx 22.5.0 上，`{workspaceRoot}` 输入无法到达哈希器：已用仓库根目录下的受跟踪对照文件验证，修改该文件并未改变任务哈希。不要再在此浪费时间；改为对消费项目关闭缓存。

### 整字段 null 与嵌套 null

整个可选字段设为 `null` 表示缺失，包括可选任意 JSON 载荷。生产者省略该字段；Schema 要求省略而非显式 `null`。此规则适用于 `rawEvent`、运行和子代理的 `result`、工具 `parameters`、恢复 `payload`、运行输入 `state` 和 `forwardedProps`，以及媒体部分的 `metadata`。每个 SDK 规则相同。

必需的任意 JSON 字段保留整个 `null` 值：`STATE_SNAPSHOT.snapshot`、`RAW.event`、`CUSTOM.value`，以及 JSON Patch 的 add、replace 或 test 操作的 `value`。`CUSTOM.value` 在 .NET 中的可空类型不改变该要求；其序列化器显式保留 null。`ACTIVITY_SNAPSHOT.content` 是必需对象，因此整个 `null` 在每个 SDK 中均无效。

嵌套在任何允许的对象或数组内的 null 仍为数据，原样保留。例如，可选 `result: null` 被省略，而 `result: {"selectedId": null}` 被保留。同样的区分适用于元数据键下的 null 以及状态对象或数组内的 null。

## `agent-capabilities.json`

`AgentCapabilities` 在 `spec/1.0/schema.json` 中定义一次，并为每个 SDK 生成。1.0 之前，每个 SDK 携带自己的手写副本，且已发生漂移：.NET 将 `identity.metadata` 和 `custom` 类型化为字典，而其他 SDK 携带开放 JSON；三者都将子代理列表拼写为 `subAgents`，而协议其余部分将该词拼写为一个词（`subagentRunId`）。该 fixture 固定生成模型的线格式，使三者不再漂移。

### 结构

```jsonc
{
  "cases": [
    {
      "name": "partial_as_a_real_producer_declares",
      "producedBy": ["typescript", "python", "dotnet"],
      "note": "why this case is here",
      "input": {
        "identity": { "type": "langgraph" },
        "transport": { "streaming": true },
      },
      "expected": {
        "identity": { "type": "langgraph" },
        "transport": { "streaming": true },
      },
    },
  ],
}
```

每个 SDK 的测试遍历 `cases`，对每个在 `producedBy` 中列出自身名称的用例：

1. 将 `input` 解析为 SDK 的 `AgentCapabilities` 模型，
2. 通过 SDK 的官方 JSON 路径重新序列化，
3. 断言结果解析后恰好等于 `expected`。

`expected` 是精确匹配，因此未设置的可选成员以 `null` 出现会导致用例失败；任何 SDK 为输入省略的组凭空造值也会失败——省略意味着**未声明**，任何 SDK 不得填充。按键开放的成员（`identity.metadata`、`custom`）下的 `null` **值**是相反情况：它是数据，协议规定必须保留，`open_values_may_be_null` 用例要求每个 SDK 原样传递。`full_every_group_populated` 用例覆盖每个字段，包括 .NET 新增的两个字段和单词形式的 `subagents` 键。全部四个文档——`full_every_group_populated`、`minimal_nothing_declared`、`partial_as_a_real_producer_declares` 和 `open_values_may_be_null`——位于 `spec/1.0/fixtures/AgentCapabilities/valid/` 下，分别为 `full.json`、`minimal.json`、`partial.json` 和 `open-values-null.json`，规范测试框架在此依据 Schema 验证它们；本文件则是各 SDK 相互对齐之处。该映射不是需要任何人记住的约定：`spec/harness/fixtures.test.ts` 双向断言它，并断言每个文档的两份副本解析为相同值（对解析后的 JSON 执行 `expect(c.input).toEqual(spec)`——因此两份副本的格式和键顺序可以不同，但值不得不同）。

### 消费者

| SDK        | 测试                                                                             |
| ---------- | -------------------------------------------------------------------------------- |
| TypeScript | `sdks/typescript/packages/core/src/__tests__/agent-capabilities-fixture.test.ts` |
| Python     | `sdks/python/tests/test_capabilities.py`                                         |
| .NET       | `sdks/dotnet/tests/AGUI.Abstractions.UnitTests/AgentCapabilitiesFixtureTest.cs`  |