# 性能指南

## K2 编译器优势

AG-UI Kotlin SDK 使用 Kotlin 2.1.21 的 K2 编译器，带来显著的性能提升：

### 编译性能
- 增量编译**快 2 倍**
- 编译期间内存占用**减少 50%**
- 类型推断改进，**IDE 响应更快**

### 运行时性能
- **协程优化**，挂起函数内联更好
- **减少分配**，Flow 操作中
- **更小的字节码**，面向多平台目标

### 二进制大小优化
| 平台 | K1 编译器 | K2 编译器 | 减少 |
|----------|-------------|-------------|-----------|
| Android  | ~450KB      | ~380KB      | 15.5%     |
| iOS      | ~520KB      | ~420KB      | 19.2%     |
| JVM      | ~380KB      | ~320KB      | 15.8%     |

## Ktor 3 改进

升级到 Ktor 3.1.3 带来：

- SSE 解析**快 30%**
- **原生 HTTP/2** 支持（可用时）
- **内存效率提升**，流式响应
- **取消处理更好**，结构化并发

## 序列化性能

kotlinx.serialization 1.8.1 提供：

- 大载荷 JSON 解析**快 2.5 倍**
- 反序列化期间内存占用**减少 50%**
- 可序列化类的**编译时验证**

## 性能最佳实践

### 1. 高效使用 Flow 操作符
```kotlin
// Good - processes items as they arrive
agent.runAgent()
    .filter { it is TextMessageContentEvent }
    .map { (it as TextMessageContentEvent).delta }
    .collect { print(it) }

// Bad - collects everything in memory
val allEvents = agent.runAgent().toList()
allEvents.filter { it is TextMessageContentEvent }
    .forEach { print((it as TextMessageContentEvent).delta) }
```

### 2. 处理背压
```kotlin
agent.runAgent()
    .buffer(capacity = 64) // Buffer events if processing is slow
    .conflate() // Drop intermediate values if needed
    .collect { handleEvent(it) }
```

### 3. 正确使用取消
```kotlin
val job = scope.launch {
    agent.runAgent().collect { event ->
        if (shouldCancel()) {
            currentCoroutineContext().cancel()
        }
        handleEvent(event)
    }
}

// Clean cancellation
job.cancelAndJoin()
```

### 4. 优化状态更新
```kotlin
// Use state snapshots for large updates
if (changedProperties > 10) {
    emit(StateSnapshotEvent(snapshot = newState))
} else {
    // Use deltas for small updates
    emit(StateDeltaEvent(delta = patches))
}
```

## 内存管理

### 事件处理
- 事件以流式处理，不加载到内存
- 使用容量有限的 `buffer()` 防止内存问题
- 在 `finally` 块中实现正确的清理

### 消息历史
- 考虑为长对话实现消息修剪
- 适当时候对缓存数据使用弱引用
- 在生产环境中使用 LeakCanary（Android）等工具监控内存使用

## 网络优化

### 连接池
```kotlin
val agent = HttpAgent(HttpAgentConfig(
    url = "https://api.example.com",
    headers = mapOf(
        "Connection" to "keep-alive",
        "Keep-Alive" to "timeout=600"
    )
))
```

### 压缩
AG-UI Kotlin SDK 在服务器支持时自动处理 gzip 压缩。

## 监控

### 性能指标
```kotlin
agent.runAgent()
    .onEach { measureTimeMillis { processEvent(it) } }
    .collect { event ->
        logger.debug { "Processed ${event.type} in ${time}ms" }
    }
```

### 资源使用
监控：
- 协程数量，使用 `kotlinx.coroutines.debug`
- 内存使用，使用平台分析器
- 网络带宽，使用 Ktor 的日志功能

## 平台特定优化

### Android
- 发布构建使用 R8/ProGuard
- 启用代码收缩和混淆
- 考虑使用基线配置文件加快启动

### iOS
- 启用 Swift/Objective-C 互操作优化
- 生产构建使用 release 模式
- 考虑使用 Kotlin/Native 内存模型注解

### JVM
- 使用适当的 GC 设置
- 启用 JIT 编译器优化
- 考虑使用 GraalVM 构建原生镜像