# QueryEngine 查询引擎设计深度分析

## 1. 架构定位

QueryEngine 是 Claude Code 的**核心驱动引擎**，管理一次完整对话会话的生命周期。它处于 REPL/Headless/SDK 三种入口层的下方，`query()` 核心循环的上方。

```
┌──────────────────────────────────────────────────────────────┐
│                      入口层                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────────┐   │
│  │ REPL (交互)  │  │ SDK (编程)  │  │ Headless (-p 模式) │   │
│  └──────┬──────┘  └──────┬──────┘  └─────────┬──────────┘   │
│         │                │                   │               │
│         └────────────────┼───────────────────┘               │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   QueryEngine                           │ │
│  │  • 会话状态管理（mutableMessages, readFileState）        │ │
│  │  • 系统提示构建                                         │ │
│  │  • 用户输入预处理（斜杠命令、附件）                      │ │
│  │  • 权限包装                                             │ │
│  │  • 成本/用量追踪                                        │ │
│  │  • 预算控制                                             │ │
│  │  • 会话持久化（Transcript）                              │ │
│  │  • 结果提取与错误处理                                   │ │
│  └───────────────────────┬─────────────────────────────────┘ │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   query() 核心循环                       │ │
│  │  API 调用 → 工具执行 → 消息追加 → 循环/终止              │ │
│  └─────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 核心设计

### 2.1 单实例持久化

```typescript
/**
 * QueryEngine owns the query lifecycle and session state for a conversation.
 * One QueryEngine per conversation. Each submitMessage() call starts a new
 * turn within the same conversation. State (messages, file cache, usage, etc.)
 * persists across turns.
 */
export class QueryEngine {
  private config: QueryEngineConfig        // 配置（工具、命令、MCP 等）
  private mutableMessages: Message[]       // 会话消息（跨轮次持久化）
  private abortController: AbortController // 中断控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage     // 累积用量
  private readFileState: FileStateCache    // 文件读取缓存
  private discoveredSkillNames: Set<string>       // 技能发现追踪
  private loadedNestedMemoryPaths: Set<string>    // 嵌套记忆路径
}
```

**设计要点**：
- **一个对话 = 一个 QueryEngine**：状态跨轮次持久化
- **每次 submitMessage() = 一个新轮次**：但在同一会话内
- **mutableMessages 是核心状态**：所有消息追加都写到这里

### 2.2 AsyncGenerator 流式输出

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown>
```

submitMessage 返回 **AsyncGenerator**，这是整个流式架构的基础：

```
QueryEngine.submitMessage()
    │
    │  yield SDKMessage ─────────────────────► 调用方（REPL/SDK）
    │
    ├── yield system_init ────► 初始化信息
    ├── yield user_replay ────► 用户消息回放
    ├── yield assistant ──────► 模型响应（流式）
    ├── yield progress ───────► 工具执行进度
    ├── yield stream_event ───► API 流事件
    ├── yield attachment ─────► 附件/结构化输出
    ├── yield compact_boundary ► 上下文压缩边界
    ├── yield tool_use_summary ► 工具使用摘要
    └── yield result ─────────► 最终结果（成功/失败/超限）
```

**为什么用 AsyncGenerator**：
1. **流式输出**：模型响应可以逐块 yield，不需要等整个回复完成
2. **背压支持**：消费方可以控制消费速度
3. **可取消**：通过 AbortController 随时中断
4. **可组合**：`yield*` 委托给子 Generator

---

## 3. submitMessage 完整流程

### 3.1 阶段总览

```
submitMessage(prompt)
│
├── 阶段1: 初始化（L220-280）
│   ├── setCwd(cwd)
│   ├── 包装 canUseTool（追踪权限拒绝）
│   ├── 解析模型和 thinking 配置
│   └── 构建系统提示
│
├── 阶段2: 用户输入处理（L310-350）
│   ├── processUserInput() 处理斜杠命令和附件
│   ├── 追加消息到 mutableMessages
│   └── 记录 Transcript（崩溃可恢复）
│
├── 阶段3: 技能和插件加载（L410-418）
│   ├── getSlashCommandToolSkills()
│   └── loadAllPluginsCacheOnly()（非阻塞）
│
├── 阶段4: yield system_init（L420）
│
├── 阶段5: 本地命令处理（shouldQuery=false）（L430-500）
│   ├── yield 本地命令输出
│   └── yield result → return
│
├── 阶段6: 核心查询循环（L530-700）
│   ├── query() AsyncGenerator
│   │   ├── API 调用
│   │   ├── 工具执行
│   │   ├── 消息追加
│   │   └── 压缩/重试
│   │
│   ├── 消息类型分发：
│   │   ├── assistant → yield 响应
│   │   ├── user → turnCount++
│   │   ├── progress → yield 进度
│   │   ├── stream_event → 追踪用量
│   │   ├── attachment → 处理结构化输出
│   │   ├── system → 处理压缩边界
│   │   └── tombstone → 跳过
│   │
│   └── 预算检查（每次迭代）
│       ├── maxBudgetUsd → 超限则终止
│       ├── maxTurns → 超限则终止
│       └── structuredOutput 重试限制
│
└── 阶段7: 结果提取（L730-800）
    ├── 提取文本结果
    ├── 判断成功/失败
    └── yield result → return
```

### 3.2 阶段1: 系统提示构建

```typescript
const {
  defaultSystemPrompt,
  userContext: baseUserContext,
  systemContext,
} = await fetchSystemPromptParts({
  tools,
  mainLoopModel: initialMainLoopModel,
  additionalWorkingDirectories: ...,
  mcpClients,
  customSystemPrompt: customPrompt,
})

// 合并 Coordinator 上下文
const userContext = {
  ...baseUserContext,
  ...getCoordinatorUserContext(mcpClients, scratchpadDir),
}

// 组装最终系统提示
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])
```

**设计亮点**：
- 系统提示通过 **`asSystemPrompt`** 包装为不可变类型
- 支持 `customSystemPrompt`（完全替换）和 `appendSystemPrompt`（追加）
- Coordinator 模式的上下文通过**依赖注入**合并（避免循环依赖）

### 3.3 阶段2: 用户输入处理

```typescript
const {
  messages: messagesFromUserInput,
  shouldQuery,           // 是否需要查询 API
  allowedTools,          // 允许的工具列表
  model: modelFromUserInput,  // 斜杠命令可能切换模型
  resultText,            // 本地命令的文本结果
} = await processUserInput({
  input: prompt,
  mode: 'prompt',
  context: processUserInputContext,
  messages: this.mutableMessages,
})

this.mutableMessages.push(...messagesFromUserInput)
```

**shouldQuery 分支**：
- `true` → 进入核心查询循环（调用 API）
- `false` → 仅处理本地斜杠命令（如 /help、/clear），直接返回结果

### 3.4 阶段3: Transcript 持久化

```typescript
// 在进入查询循环前写入 Transcript
// 确保进程崩溃后可通过 --resume 恢复
if (persistSession && messagesFromUserInput.length > 0) {
  const transcriptPromise = recordTranscript(messages)
  if (isBareMode()) {
    void transcriptPromise  // --bare 模式：fire-and-forget
  } else {
    await transcriptPromise  // 正常模式：等待写入完成
  }
}
```

**设计考量**：
- 用户消息在 API 调用**之前**写入磁盘
- 如果 API 调用期间进程被杀，`--resume` 可以恢复
- `--bare` 模式（脚本调用）不阻塞等待写入，节省 ~4-30ms

### 3.5 阶段6: 核心查询循环

```typescript
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel,
  querySource: 'sdk',
  maxTurns,
  taskBudget,
})) {
  // 消息类型分发处理
  switch (message.type) {
    case 'assistant': ...
    case 'user': ...
    case 'progress': ...
    case 'stream_event': ...
    case 'attachment': ...
    case 'system': ...
    case 'tombstone': ...
    case 'tool_use_summary': ...
  }

  // 预算检查（每次迭代）
  if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
    yield { type: 'result', subtype: 'error_max_budget_usd', ... }
    return
  }
}
```

---

## 4. 消息类型与生命周期

### 4.1 消息流图

```
query() yield
    │
    ├── stream_event (message_start)
    │   └── 重置 currentMessageUsage
    │
    ├── assistant ──► yield 到调用方
    │   ├── text content
    │   ├── tool_use content
    │   └── thinking content
    │
    ├── stream_event (message_delta)
    │   └── 更新 currentMessageUsage + stop_reason
    │
    ├── progress ──► yield 到调用方
    │   └── 工具执行进度
    │
    ├── user ──► yield 到调用方 + turnCount++
    │   └── tool_result 或用户输入
    │
    ├── attachment ──► 处理特殊信号
    │   ├── structured_output → 提取结构化输出
    │   ├── max_turns_reached → yield error + return
    │   └── queued_command → yield user replay
    │
    ├── stream_event (message_stop)
    │   └── 累加 currentMessageUsage 到 totalUsage
    │
    ├── system
    │   ├── compact_boundary → 压缩边界处理
    │   ├── api_error → yield retry 通知
    │   └── snip_boundary → 历史裁剪
    │
    ├── tool_use_summary ──► yield 到调用方
    │
    └── tombstone ──► 跳过（控制信号）
```

### 4.2 用量追踪机制

```typescript
let currentMessageUsage: NonNullableUsage = EMPTY_USAGE

// message_start: 重置
if (message.event.type === 'message_start') {
  currentMessageUsage = EMPTY_USAGE
  currentMessageUsage = updateUsage(currentMessageUsage, message.event.message.usage)
}

// message_delta: 更新
if (message.event.type === 'message_delta') {
  currentMessageUsage = updateUsage(currentMessageUsage, message.event.usage)
}

// message_stop: 累加到总计
if (message.event.type === 'message_stop') {
  this.totalUsage = accumulateUsage(this.totalUsage, currentMessageUsage)
}
```

---

## 5. 三重预算控制

### 5.1 maxTurns（轮次限制）

```typescript
// 在 query() 内部通过 attachment 信号传递
if (message.attachment.type === 'max_turns_reached') {
  yield { type: 'result', subtype: 'error_max_turns', ... }
  return
}
```

### 5.2 maxBudgetUsd（美元预算）

```typescript
// 每次循环迭代检查
if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
  yield { type: 'result', subtype: 'error_max_budget_usd', ... }
  return
}
```

### 5.3 structuredOutput 重试限制

```typescript
// 只在 user 消息时检查（避免过于频繁）
if (message.type === 'user' && jsonSchema) {
  const currentCalls = countToolCalls(this.mutableMessages, SYNTHETIC_OUTPUT_TOOL_NAME)
  const callsThisQuery = currentCalls - initialStructuredOutputCalls
  const maxRetries = parseInt(process.env.MAX_STRUCTURED_OUTPUT_RETRIES || '5', 10)
  if (callsThisQuery >= maxRetries) {
    yield { type: 'result', subtype: 'error_max_structured_output_retries', ... }
    return
  }
}
```

---

## 6. 权限包装

```typescript
// 包装 canUseTool 以追踪权限拒绝
const wrappedCanUseTool: CanUseToolFn = async (
  tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision,
) => {
  const result = await canUseTool(tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision)

  // 追踪拒绝记录用于 SDK 报告
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }

  return result
}
```

**设计要点**：
- 原始 `canUseTool` 不被修改
- 包装层透明地追踪权限拒绝
- 最终 result 消息包含所有 `permission_denials`

---

## 7. 上下文压缩（Compact）

### 7.1 Compact Boundary 处理

```typescript
case 'system': {
  if (message.subtype === 'compact_boundary' && message.compactMetadata) {
    // 释放压缩前的消息，允许 GC
    const mutableBoundaryIdx = this.mutableMessages.length - 1
    if (mutableBoundaryIdx > 0) {
      this.mutableMessages.splice(0, mutableBoundaryIdx)
    }
    const localBoundaryIdx = messages.length - 1
    if (localBoundaryIdx > 0) {
      messages.splice(0, localBoundaryIdx)
    }
    // yield 边界消息给 SDK
    yield { type: 'system', subtype: 'compact_boundary', ... }
  }
}
```

### 7.2 Snip 压缩（History Snip）

```typescript
// 通过依赖注入的 snipReplay 回调处理
// 避免在 QueryEngine 中引入 feature-gated 字符串
const snipResult = this.config.snipReplay?.(message, this.mutableMessages)
if (snipResult !== undefined) {
  if (snipResult.executed) {
    this.mutableMessages.length = 0
    this.mutableMessages.push(...snipResult.messages)
  }
  break
}
```

**设计亮点**：
- Snip 逻辑通过**回调注入**，QueryEngine 本身不含 feature-gated 字符串
- 可测试性：测试时不需要启用 feature flag

---

## 8. ask() 便捷函数

```typescript
/**
 * Convenience wrapper around QueryEngine for one-shot usage.
 */
export async function* ask({ ... }): AsyncGenerator<SDKMessage, void, unknown> {
  const engine = new QueryEngine({ ... })

  try {
    yield* engine.submitMessage(prompt, { uuid: promptUuid, isMeta })
  } finally {
    setReadFileCache(engine.getReadFileState())
  }
}
```

**ask() vs QueryEngine**：
- `ask()` 是一次性使用：创建 Engine → 提交消息 → 返回
- `QueryEngine` 支持多次 `submitMessage()`（跨轮次会话）
- `ask()` 在 finally 中保存 readFileState

---

## 9. 错误处理与诊断

### 9.1 error_during_execution 诊断

```typescript
if (!isResultSuccessful(result, lastStopReason)) {
  yield {
    type: 'result',
    subtype: 'error_during_execution',
    errors: (() => {
      const all = getInMemoryErrors()
      const start = errorLogWatermark
        ? all.lastIndexOf(errorLogWatermark) + 1
        : 0
      return [
        `[ede_diagnostic] result_type=${edeResultType} last_content_type=${edeLastContentType} stop_reason=${lastStopReason}`,
        ...all.slice(start).map(_ => _.error),
      ]
    })(),
  }
  return
}
```

**诊断信息包含**：
- `result_type`：最终消息类型（应该是 assistant 或 user）
- `last_content_type`：最后内容块类型
- `stop_reason`：模型停止原因
- `errors[]`：本轮的错误日志（基于 watermark，不包含历史错误）

### 9.2 API 错误重试通知

```typescript
if (message.subtype === 'api_error') {
  yield {
    type: 'system',
    subtype: 'api_retry',
    attempt: message.retryAttempt,
    max_retries: message.maxRetries,
    retry_delay_ms: message.retryInMs,
    error_status: message.error.status ?? null,
    error: categorizeRetryableAPIError(message.error),
  }
}
```

---

## 10. 设计模式总结

| 模式 | 应用 | 价值 |
|------|------|------|
| **AsyncGenerator** | submitMessage 流式输出 | 流式、背压、可取消、可组合 |
| **单实例持久化** | 一个 Engine = 一个会话 | 状态跨轮次保持 |
| **装饰器模式** | wrappedCanUseTool | 透明追踪权限拒绝 |
| **依赖注入** | snipReplay、coordinator context | 解耦 feature-gated 逻辑 |
| **策略模式** | shouldQuery 分支 | 本地命令 vs API 查询 |
| **观察者模式** | 用量追踪 + 预算检查 | 实时监控资源消耗 |
| **Template Method** | 系统提示组装 | 灵活的 Prompt 构建 |
| **双路径** | REPL 交互 vs SDK/Headless | 同一引擎支持多种入口 |

---

## 11. 可借鉴要点

1. **AsyncGenerator 作为核心接口**：任何需要流式输出的 AI 系统都应该使用 AsyncGenerator，它天然支持流式、背压和取消

2. **三重预算控制**：轮次限制 + 美元预算 + 结构化输出重试，在生产环境中缺一不可

3. **Transcript 先写策略**：用户消息在 API 调用前写入磁盘，确保崩溃可恢复

4. **权限包装器**：不修改原始权限函数，通过装饰器模式透明追踪

5. **Feature Flag 通过依赖注入**：核心引擎不包含 feature-gated 代码，通过回调注入保持可测试性

6. **消息类型统一分发**：所有消息（assistant、user、progress、system、attachment 等）通过统一的 switch 分发处理，易于扩展

7. **用量追踪的 message_start/delta/stop 模式**：与 LLM API 的流式协议对齐，精准追踪每条消息的 token 消耗
