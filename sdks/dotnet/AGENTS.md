# AG-UI .NET SDK - 编码说明

设计理念、包结构以及各子系统如何协同工作，请参阅 `docs/architecture.md`。

## 前置条件

- .NET 10 SDK（确切版本见 `global.json`；已配置 `rollForward: minor`）。
- 以下所有命令均在 `sdks/dotnet/` 目录下运行。

### 配置仓库本地 SDK（可选，隔离环境）

如需使用精确锁定的 SDK 构建而不影响机器全局安装，请使用配置脚本。脚本从 `global.json` 下载 SDK，以及多目标测试项目所需的 .NET 8 和 9 运行时，放入被 git 忽略的 `.dotnet/` 文件夹，并仅针对它构建/测试（`DOTNET_MULTILEVEL_LOOKUP=0`）：

```bash
./build.cmd            # Windows：配置 + 构建
./build.sh             # Linux/macOS：配置 + 构建
./build.sh --test      # 配置 + 测试
```

`eng/install-dotnet.ps1` / `eng/install-dotnet.sh` 仅执行配置步骤，接受 `-ExtraChannel`/`--extra-channel`（或 `-ExtraVersion`/`--extra-version`）以并行安装额外的 SDK（例如 .NET 11 预览版）。

## 构建

```bash
dotnet build
```

解决方案文件为 `AGUI.slnx`。`Directory.Build.props` 将 `LangVersion` 设为 `latest`，启用可空引用类型，并将警告视为错误。`Directory.Packages.props` 集中管理所有 NuGet 版本（中央包管理）。`Directory.Build.targets` 在项目中存在 `PublicAPI.Shipped.txt` 时条件启用 `PublicApiAnalyzers`。

## 运行测试

```bash
dotnet test
```

运行解决方案中所有单元测试和集成测试项目。开发时如需更快反馈，可针对单个项目运行，如下所述。

### 单元测试

每个 `src/` 包对应一个单元测试项目：

| 项目 | 覆盖内容 | 关键模式 |
|---|---|---|
| `tests/AGUI.Abstractions.UnitTests/` | 事件序列化往返、针对 TypeScript JSON 夹具的向后兼容性 | `JsonDocument` 属性断言、`FixtureLoader` 加载 `Compatibility/` 中的跨 SDK 夹具 |
| `tests/AGUI.Client.UnitTests/` | 客户端构建器、协议规则、传输协商 | 标准 xunit 断言 |
| `tests/AGUI.Formatting.UnitTests/` | `SseEventStreamFormatter` 读写往返、SSE 线格式 | 标准 xunit 断言 |
| `tests/AGUI.Protobuf.UnitTests/` | Protobuf 编解码器、`ProtobufEventStreamFormatter`、`JsonElement`↔`google.protobuf.Value` 桥接 | 标准 xunit 断言 |
| `tests/AGUI.Server.UnitTests/` | `ChatResponseUpdate` → AG-UI 事件转换、混合工具调用、中断内容 | 标准 xunit 断言 |

运行单个单元测试项目：

```bash
dotnet test tests/AGUI.Abstractions.UnitTests/
```

测试文件直接位于测试项目根目录下（不镜像到 `Events/` 子文件夹）。事件测试的标准模式：

```csharp
[Fact]
public void Serialization_RoundTrips()
{
    var evt = new RunStartedEvent { ThreadId = "t1", RunId = "r1", Timestamp = 1234567890 };

    var json = JsonSerializer.Serialize(evt, AGUIJsonSerializerContext.Default.RunStartedEvent);
    using var doc = JsonDocument.Parse(json);

    Assert.Equal("RUN_STARTED", doc.RootElement.GetProperty("type").GetString());
    Assert.Equal("t1", doc.RootElement.GetProperty("threadId").GetString());
}
```

通过 `JsonDocument` 解析来验证具体的 JSON 属性名。不要只对反序列化后的对象做断言——那无法发现命名错误。

#### 向后兼容性测试

`tests/AGUI.Abstractions.UnitTests/Compatibility/` 包含由 TypeScript 参考实现生成的 JSON 夹具文件。测试将这些夹具反序列化为 .NET 类型并验证值匹配。这能发现实现之间的线格式漂移。使用 `FixtureLoader` 加载夹具数组。每个测试方法覆盖一种事件形态。

### 集成测试

`tests/AGUI.Hosting.AspNetCore.IntegrationTests/` 测试完整 HTTP 流水线——提交 `RunAgentInput`、通过网络流式传输事件，并通过原始事件流和 `AGUIChatClient`（`IChatClient`）抽象验证结果。（尽管服务端包已重命名为 `AGUI.Server`，项目名仍有意保留。）测试使用 `WebApplicationFactory<TProgram>` 和 `ConfigureTestServices` 注入 `DelegatingStreamingChatClient`（基于 `Func` 的 `IChatClient`）。测试基础设施支持录制和回放 `ChatResponseUpdate` 序列，使测试无需调用真实 LLM 即可确定性运行。每个测试是一个 `[Theory]`，按 `TransportFormat`（`Json` 和 `Protobuf`）参数化；两种传输解码为相同的流，因此跨格式复用相同的 Verify 基线。

运行集成测试：

```bash
dotnet test tests/AGUI.Hosting.AspNetCore.IntegrationTests/
```

集成测试项目引用每个 `samples/GettingStarted/Step*` 项目。`Samples/GettingStarted/` 子文件夹包含的测试将每个示例作为真实主机启动，通过 `FakeChatClient` 回放预录的 `ChatResponseUpdate` 序列，并使用 `Verify.Xunit` 快照文件（`.verified.txt`）验证输出。快照测试失败时，使用 `--environment VERIFY_ACCEPT=true` 运行或查看 `.received.txt` 差异。

### 测试中不应做的事

- 不要用反射枚举类型或验证成员关系。
- 不要通过比较完整 JSON 字符串来断言行为（脆弱）。用 `JsonDocument` 解析并检查单个属性。

## 运行示例

`samples/GettingStarted/` 下的每个示例（Step01 到 Step14）是一对 Server/Client。服务端以 ASP.NET Core 应用托管 AG-UI 端点（ASP.NET 胶水代码来自共享的 `samples/AGUI.Samples.Shared` 项目）；客户端通过 `AGUIChatClient` 驱动它。手动运行示例服务端：

```bash
dotnet run --project samples/GettingStarted/Step01_GettingStarted/Step01_GettingStarted.Server/
```

`samples/AGUIClientServer/` 目录包含一个完整的 Dojo 服务端，含多个智能体场景。

## 项目布局

- `src/AGUI.Abstractions/` — 协议类型：事件、消息、工具、能力、序列化上下文（`AGUIJsonSerializerContext`）以及 `AGUIJsonUtilities.RegisterInterruptContentTypes`。
- `src/AGUI.Formatting/` — 线格式格式化器：`IAGUIEventStreamFormatter`（双向读写）和 `SseEventStreamFormatter`（SSE 线格式）。依赖 Abstractions + `System.Net.ServerSentEvents`。
- `src/AGUI.Protobuf/` — Protobuf 编解码器（`internal` 的 `AGUIProtobuf`）、公开的 `ProtobufEventStreamFormatter` 以及 `JsonElement`↔`google.protobuf.Value` 桥接。生成的 proto 类型为 `internal`；`.proto` Schema 引用自 `sdks/typescript/packages/proto`（未复制）。依赖 Abstractions + Formatting + `Google.Protobuf`。
- `src/AGUI.Client/` — `AGUIChatClient`（`IChatClient`，由 `AGUIChatClientOptions` 构造）、`AGUIHttpTransport`/`IAGUITransport`，以及公开的协商原语（`AGUIEventStreamHandler` `DelegatingHandler` + `ReadAGUIEventStreamAsync`），调用方可将其接入自己的 `HttpClient` 以请求 protobuf。依赖 Formatting。
- `src/AGUI.Server/` — 框架无关的服务端适配器（无 ASP.NET）：`RunAgentInputExtensions.ToChatRequestContext`、`ChatRequestContext`、`ChatResponseUpdateAGUIExtensions.AsAGUIEventStreamAsync`、流式 `AGUIStreamOptions`、`AGUIConstants`。依赖 Abstractions + `M.E.AI.Abstractions`。
- `samples/AGUI.Samples.Shared/` — 唯一的 ASP.NET 项目（`FrameworkReference Microsoft.AspNetCore.App`）。承载 ASP.NET 胶水代码：`AGUIResults`（协商式 `IResult`）、`AGUIEventStreamResult`、`MapAGUI` 端点扩展以及 `AddAGUI` DI 注册。
- `tests/AGUI.Abstractions.UnitTests/` — 序列化往返和向后兼容性测试。
- `tests/AGUI.Client.UnitTests/` — 客户端构建器、协议和传输协商测试。
- `tests/AGUI.Formatting.UnitTests/` — SSE 格式化器往返测试。
- `tests/AGUI.Protobuf.UnitTests/` — Protobuf 编解码器和格式化器测试。
- `tests/AGUI.Server.UnitTests/` — 流转换单元测试。
- `tests/AGUI.Hosting.AspNetCore.IntegrationTests/` — 使用 `WebApplicationFactory` 的端到端测试，按 `TransportFormat` 参数化，包括示例回放测试（`AGUI.Server` 重命名后名称保留）。
- `samples/GettingStarted/` — 渐进式 Server/Client 示例对（Step01–Step14；Step12 = 并行工具调用，Step13 = protobuf，Step14 = OpenTelemetry 追踪）。
- `samples/AGUIClientServer/` — 完整的 Dojo 服务端，含多个智能体场景。

## 端点模式

每个 AG-UI 端点遵循此形态（GettingStarted 示例通过 `samples/AGUI.Samples.Shared` 中的共享 `app.MapAGUI("/")` 辅助方法映射）：

1. `MapPost(pattern, handler)` — 接收 `[FromBody] RunAgentInput`。
2. 用 `var ctx = input.ToChatRequestContext(jsonSerializerOptions, streamOptions?)` 适配到 MEAI。返回的 `ChatRequestContext` 携带转换后的 `ChatMessage` 列表和配置好的 `ChatOptions`（输入存于 `AdditionalProperties[AGUIConstants.RunAgentInputKey]`，客户端工具已通过审批流流水线路由）。
3. 调用 `chatClient.GetStreamingResponseAsync(ctx.Messages, ctx.ChatOptions, cancellationToken)`。
4. 通过 `.AsAGUIEventStreamAsync(ctx, cancellationToken)` 管道获取 AG-UI 事件流。
5. 返回 `AGUIResults.Events(events, httpContext, cancellationToken)`（来自 `AGUI.Samples.Shared`）。这个协商式 `IResult` 检查请求的 `Accept` 头，将流编码为 Server-Sent Events（默认）或 protobuf（当服务端将 `ProtobufEventStreamFormatter` 注册为 `IAGUIEventStreamFormatter` 且客户端接受时）。端点不再手写 `TypedResults.ServerSentEvents(...)`。

如果端点需要框架特定的内容映射（例如推理、自定义工作流事件）或自定义中断分类器，请通过流式 `MapContent(...)` / `MapInterrupt(...)` / `MapCall(...)` / `MapResult(...)` 调用在传给 `ToChatRequestContext` 的 `AGUIStreamOptions` 实例上配置它们。

## 公开 API 表面

每个 `src/` 项目都有由 `Microsoft.CodeAnalysis.PublicApiAnalyzers` 管理的 `PublicAPI.Shipped.txt` 和 `PublicAPI.Unshipped.txt`。添加或更改公开成员时，更新 `PublicAPI.Unshipped.txt`。忘记更新会导致构建失败。

## JSON 序列化

每个协议类型必须 AOT 兼容。规则：

- 为每个新类型在 `AGUIJsonSerializerContext` 上添加 `[JsonSerializable(typeof(T))]`。
- 在每个序列化属性上使用 `[JsonPropertyName("camelCase")]`。上下文也设置了 `PropertyNamingPolicy = CamelCase`，但为清晰和 PublicAPI 分析器兼容性，仍需显式特性。
- 线类型的每个可空属性携带自己的忽略条件，通常为
  `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)]`。规范生成器在生成的类型上发出它；在手写类型上手动添加。`AGUIJsonSerializerContext` 上的 `DefaultIgnoreCondition = WhenWritingNull` 本身不够：它存在于上下文自己的选项上，不会随上下文进入主机的 `JsonSerializerOptions`。将裸上下文插入其解析器链的主机（Microsoft Agent Framework 的 `ConfigureAGUIJsonOptions` 就是这样）否则会写出 `"parentRunId": null` 之类，而 1.0 接收方会拒绝。`NullOmissionTest` 通过此类主机拥有的选项扫描每个线类型，并在缺少特性时失败。作为值的 `null`（状态内、补丁、元数据、`CUSTOM.value`）不受影响：该特性只丢弃属性自身的 null。
- 契约允许生产者省略的非可空 `JsonElement` 需要
  `[JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]` 代替——它永不为 `null`，所以 null 规则无法触及它，而未设置的 `JsonElement` 根本无法写出。参见 `AGUITool.Parameters` 和 `RunAgentInput`。
- 将 AG-UI 类型组合到你自己的 `JsonSerializerOptions` 时，插入
  `AGUIJsonUtilities.DefaultTypeInfoResolver`（在链的前端，位于任何为任意类型应答的解析器之前），而非 `AGUIJsonSerializerContext.Default`。
- 将必需的字符串属性初始化为 `string.Empty`。将集合初始化为 `[]`。
- 多态类型使用基于鉴别器属性的手写 `JsonConverter<T>`（参见 `BaseEventJsonConverter`、`AGUIMessageJsonConverter`、`AGUIInputContentJsonConverter`）。
- 通过源生成上下文序列化：`AGUIJsonSerializerContext.Default.{TypeName}`。
- 绝不要使用 `JsonSerializer.Serialize<object>(...)` 或未经解析直接传递原始字符串。

### 添加新事件类型

1. 在 `src/AGUI.Abstractions/Events/` 中创建类，继承自 `BaseEvent`。
2. 重写 `Type` 返回 `AGUIEventTypes` 中的常量。
3. 将常量添加到 `AGUIEventTypes`。
4. 将 `[JsonSerializable(typeof(T))]` 添加到 `AGUIJsonSerializerContext`。
5. 在 `BaseEventJsonConverter` 中添加读/写分支。
6. 将类型签名添加到 `PublicAPI.Unshipped.txt`。
7. 在 `tests/AGUI.Abstractions.UnitTests/` 中编写序列化往返测试。

## 代码风格

- 每个文件一个类。文件名与类型名匹配。
- 每个非抽象类加 `sealed`。
- 不用 `record` 类型。使用带属性的 `sealed class`。
- 公开 API 中不用元组。定义命名类型。
- `if`、`for`、`foreach`、`while` 等始终使用大括号。
- `internal` 或 `private` 成员不加 XML 文档（`///`）。
- 库代码中所有 `await` 调用加 `ConfigureAwait(false)`。
- `IAsyncEnumerable` 方法中的 `CancellationToken` 参数加 `[EnumeratorCancellation]`。
- 公开 API 参数验证使用 `ArgumentNullThrowHelper.ThrowIfNull(...)`。它在现代目标上映射到 BCL 的 `ArgumentNullException.ThrowIfNull`，在 `netstandard2.0`/`net472` 上映射到手动抛出。它和 C# 编译器特性 polyfill（`init`、`required`）位于 `src/Shared/`，链接到每个多目标项目（降级 polyfill 仅为 `netstandard2.0`/`net472` 条件编译）。

## 命名

- 事件类：`{Name}Event`（`RunStartedEvent`、`TextMessageContentEvent`）。
- 事件类型鉴别器：`AGUIEventTypes` 中的 `SCREAMING_SNAKE_CASE` 字符串常量（`"RUN_STARTED"`、`"TEXT_MESSAGE_START"`）。C# 成员名使用 PascalCase（`AGUIEventTypes.RunStarted`）。
- 结果和角色常量：专用静态类中的小写字符串常量（`RunFinishedOutcome.Interrupt = "interrupt"`、`AGUIRoles.Assistant = "assistant"`）。绝不用枚举。
- 选项类：`AGUI{Purpose}Options`（`AGUIStreamOptions`）。
- 扩展类：`{Target}Extensions`（`ChatResponseUpdateAGUIExtensions`、`AGUIToolExtensions`）。
- 测试类：`{TypeUnderTest}Test`（`RunStartedEventTest`、`ChatResponseUpdateAGUIExtensionsTest`）。
- 兼容性测试类：`Compatibility/` 子文件夹中的 `{Category}CompatibilityTest`。
- DI 扩展的命名空间：`Microsoft.Extensions.DependencyInjection`。
- 所有其他类型的命名空间：匹配 `.csproj` 中的 `<RootNamespace>`（例如 `AGUI.Abstractions`、`AGUI.Server`）。无子命名空间——`Events/`、`Messages/`、`Capabilities/` 是文件夹，不是命名空间段。