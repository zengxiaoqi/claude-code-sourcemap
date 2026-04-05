# query() 核心循环设计深度分析

## 1. 架构定位

`query()` 是 Claude Code 的**心脏**——一个 `while(true)` 的无限循环，驱动 API 调用 → 工具执行 → 消息追加 → 循环/终止的完整生命周期。

```
┌──────────────────────────────────────────────────────────────────────┐
│                        query() 在架构中的位置                          │
│                                                                      │
│  QueryEngine.submitMessage()                                        │
│       │                                                              │
│       │  for await (message of query({...}))                         │
│       ▼                                                              │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      query() 核心循环                        │   │
│  │                                                              │   │
│  │  ┌─────────┐   ┌───────────┐   ┌──────────┐   ┌─────────┐  │   │
│  │  │ 上下文   │──▶│ API 调用  │──▶│ 工具执行  │──▶│ 终止?   │  │   │
│  │  │ 预处理   │   │ (流式)    │   │ (并行)   │   │ 否→循环 │  │   │
│  │  └─────────┘   └───────────┘   └──────────┘   └─────────┘  │   │
│  │       ▲                                          │ 是       │   │
│  │       └──────────────────────────────────────────┘ return   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│       │                                                              │
│       │  yield Message / StreamEvent / Terminal                     │
│       ▼                                                              │
│  QueryEngine 分发到 SDK/REPL/Headless                               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 签名与类型

```typescript
export type QueryParams = {
  messages: Message[]              // 当前会话消息
  systemPrompt: SystemPrompt       // 系统提示（不可变）
  userContext: { [k: string]: string }  // 用户上下文注入
  systemContext: { [k: string]: string } // 系统上下文追加
  canUseTool: CanUseToolFn         // 权限检查函数
  toolUseContext: ToolUseContext    // 工具执行上下文
  fallbackModel?: string           // 备用模型
  querySource: QuerySource         // 调用来源标识
  maxTurns?: number                // 最大轮次
  taskBudget?: { total: number }   // API task_budget
  deps?: QueryDeps                 // 依赖注入（测试用）
}

export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | Message | TombstoneMessage | ToolUseSummaryMessage, Terminal>
```

**返回类型**：
- **yield**：`StreamEvent | Message | TombstoneMessage | ToolUseSummaryMessage`
- **return**：`Terminal`（终止原因，如 `{ reason: 'completed' }`）

---

## 3. State 状态机

循环状态通过**单一 State 对象**管理，每次迭代顶部解构：

```typescript
type State = {
  messages: Message[]                        // 消息列表
  toolUseContext: ToolUseContext              // 工具上下文
  autoCompactTracking: AutoCompactTrackingState | undefined  // 压缩追踪
  maxOutputTokensRecoveryCount: number       // 输出 token 恢复计数
  hasAttemptedReactiveCompact: boolean       // 是否尝试过响应式压缩
  maxOutputTokensOverride: number | undefined // 输出 token 覆盖
  pendingToolUseSummary: Promise<...> | undefined  // 待处理的工具摘要
  stopHookActive: boolean | undefined        // 停止钩子状态
  turnCount: number                          // 轮次计数
  transition: Continue | undefined           // 上次继续的原因
}
```

**状态更新模式**：
```typescript
// 所有 continue 点通过赋值整个 State 对象更新
state = {
  messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
  toolUseContext,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  // ...
}
```

---

## 4. 单次迭代完整流程

```
while (true) {
│
├── ① 解构 State
│   ├── messages, toolUseContext, turnCount, ...
│   └── 技能预取（异步，不阻塞）
│
├── ② yield stream_request_start
│
├── ③ 上下文预处理管线（5步）
│   ├── 3a. applyToolResultBudget — 工具结果预算
│   ├── 3b. snipCompactIfNeeded — 历史裁剪
│   ├── 3c. microcompact — 微压缩
│   ├── 3d. contextCollapse — 上下文折叠
│   └── 3e. autocompact — 自动压缩
│
├── ④ Token 限制检查
│   └── isAtBlockingLimit → yield error + return
│
├── ⑤ API 调用（流式）
│   ├── deps.callModel() → AsyncGenerator
│   ├── 逐消息 yield（assistant/stream_event/progress）
│   ├── 收集 tool_use blocks → needsFollowUp
│   ├── 流式工具执行（StreamingToolExecutor）
│   ├── FallbackTriggeredError → 切换模型重试
│   └── Withheld 机制（不 yield 可恢复错误）
│
├── ⑥ Post-sampling Hooks
│
├── ⑦ 中断处理（abortController）
│
├── ⑧ 终止判断
│   ├── needsFollowUp === false →
│   │   ├── witheld 413 → collapse drain / reactive compact
│   │   ├── witheld max_output_tokens → escalate / recovery
│   │   ├── API error → return
│   │   ├── stop hooks → blocking / prevent
│   │   ├── token budget → continuation
│   │   └── 正常完成 → return { reason: 'completed' }
│   │
│   └── needsFollowUp === true →
│       ├── ⑨ 工具执行（runTools / StreamingToolExecutor）
│       ├── ⑩ 工具摘要生成（异步）
│       ├── ⑪ 中断处理（abort mid-tools）
│       ├── ⑫ 附件消息注入（记忆预取、文件变更、队列命令）
│       ├── ⑬ maxTurns 检查
│       └── ⑭ state = next → continue（回到 while 顶部）
}
```

---

## 5. 上下文预处理管线（5步）

每次迭代进入 API 调用前，依次执行 5 个上下文优化步骤：

```
原始消息
    │
    ▼
① applyToolResultBudget     修剪过大的工具结果
    │
    ▼
② snipCompactIfNeeded       裁剪过旧的历史消息
    │                        （记录 snipTokensFreed）
    ▼
③ microcompact              微压缩：替换冗长的工具输出
    │                        （延迟到 API 响应后 yield）
    ▼
④ contextCollapse           上下文折叠：合并相似内容
    │                        （纯投影，不修改原消息）
    ▼
⑤ autocompact               自动压缩：调用 API 生成摘要
    │                        （触发 compact_boundary）
    ▼
处理后消息 → 发送给 API
```

**每步的设计理由**：
- **Tool Result Budget**：先修剪，后续步骤看到的数据更小
- **Snip**：在 microcompact 前执行，释放的 token 数可以传递给 autocompact 的阈值检查
- **Microcompact**：在 autocompact 前，能用更轻量的方式减少 token 就不需要完整压缩
- **Context Collapse**：折叠后可能低于 autocompact 阈值，避免触发完整压缩
- **Autocompact**：最后的完整压缩，仅在以上步骤不足时触发

### 5.1 Autocompact 详情

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  { systemPrompt, userContext, systemContext, ... },
  querySource,
  tracking,
  snipTokensFreed,  // 从 snip 传递释放的 token 数
)

if (compactionResult) {
  // task_budget: 捕获压缩前的上下文窗口大小
  if (params.taskBudget) {
    const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
    taskBudgetRemaining = Math.max(0, (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext)
  }

  // 重置追踪状态
  tracking = { compacted: true, turnId: deps.uuid(), turnCounter: 0, consecutiveFailures: 0 }

  // 构建压缩后消息
  const postCompactMessages = buildPostCompactMessages(compactionResult)
  for (const message of postCompactMessages) {
    yield message  // yield compact_boundary 等消息
  }
  messagesForQuery = postCompactMessages
}
```

---

## 6. API 调用与流式处理

### 6.1 流式 API 循环

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: ...,
  tools: ...,
  signal: toolUseContext.abortController.signal,
  options: {
    model: currentModel,
    fallbackModel,
    taskBudget: params.taskBudget ? { total, remaining: taskBudgetRemaining } : undefined,
    ...
  },
})) {
  // 处理每条流式消息
}
```

### 6.2 Withheld 机制

**关键设计**：某些错误消息**不立即 yield**，而是先尝试恢复：

```typescript
// 可恢复的错误类型：
// 1. prompt_too_long (413) → 尝试 collapse drain / reactive compact
// 2. max_output_tokens → 尝试 escalate / recovery
// 3. media_size_error → 尝试 reactive compact strip-retry

let withheld = false

// Context Collapse 的 withold
if (feature('CONTEXT_COLLAPSE') && contextCollapse?.isWithheldPromptTooLong(message, ...)) {
  withheld = true
}

// Reactive Compact 的 withold
if (reactiveCompact?.isWithheldPromptTooLong(message)) {
  withheld = true
}

// Max output tokens 的 withold
if (isWithheldMaxOutputTokens(message)) {
  withheld = true
}

if (!withheld) {
  yield message  // 只有不可恢复的消息才立即 yield
}

// 消息仍然推入 assistantMessages，用于后续恢复检查
if (message.type === 'assistant') {
  assistantMessages.push(message)
}
```

### 6.3 Model Fallback

```typescript
catch (innerError) {
  if (innerError instanceof FallbackTriggeredError && fallbackModel) {
    currentModel = fallbackModel
    attemptWithFallback = true

    // 清理状态，准备重试
    yield* yieldMissingToolResultBlocks(assistantMessages, 'Model fallback triggered')
    assistantMessages.length = 0
    toolUseBlocks.length = 0

    // 刷新 StreamingToolExecutor
    streamingToolExecutor.discard()
    streamingToolExecutor = new StreamingToolExecutor(...)

    // 剥离 thinking signatures（不同模型不兼容）
    messagesForQuery = stripSignatureBlocks(messagesForQuery)

    yield createSystemMessage(`Switched to ${fallbackModel} due to high demand`)
    continue  // 重试
  }
  throw innerError
}
```

### 6.4 流式工具执行

```typescript
// StreamingToolExecutor 在流式接收 tool_use 块时就开始执行工具
if (streamingToolExecutor && !toolUseContext.abortController.signal.aborted) {
  for (const toolBlock of msgToolUseBlocks) {
    streamingToolExecutor.addTool(toolBlock, message)  // 立即开始执行
  }
}

// 获取已完成的结果
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message  // 流式 yield 工具结果
    toolResults.push(...)
  }
}
```

**价值**：不用等所有 tool_use 块到齐，第一个工具完成时就开始 yield 结果，用户体验更流畅。

---

## 7. 终止判断矩阵

当 `needsFollowUp === false`（模型没有请求工具调用）时，进入终止判断：

```
needsFollowUp === false
    │
    ├── witheld 413 (prompt_too_long)?
    │   ├── 是 → collapse drain → continue
    │   │        还 413? → reactive compact → continue
    │   │        还 413? → yield error + return
    │   └── 否 ↓
    │
    ├── witheld max_output_tokens?
    │   ├── 是 → escalate (8k→64k) → continue
    │   │        还是超? → recovery message → continue (最多3次)
    │   │        恢复失败? → yield error
    │   └── 否 ↓
    │
    ├── API error (rate limit / auth)?
    │   └── 是 → executeStopFailureHooks + return
    │
    ├── Stop hooks?
    │   ├── preventContinuation → return
    │   ├── blockingErrors → 注入错误 → continue
    │   └── 通过 ↓
    │
    ├── Token budget?
    │   ├── action === 'continue' → 注入 nudge → continue
    │   └── 通过 ↓
    │
    └── 正常完成 → return { reason: 'completed' }
```

---

## 8. 工具执行

### 8.1 双模式执行

```typescript
// 模式1：StreamingToolExecutor（流式并行）
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  // 模式2：runTools（传统顺序）
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message
    toolResults.push(...)
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

### 8.2 工具摘要生成

```typescript
// 异步生成工具使用摘要（不阻塞下一轮 API 调用）
nextPendingToolUseSummary = generateToolUseSummary({
  tools: toolInfoForSummary,
  signal: toolUseContext.abortController.signal,
}).then(summary => summary ? createToolUseSummaryMessage(summary, toolUseIds) : null)
```

---

## 9. 附件注入（轮次间）

工具执行完成后，在进入下一轮前注入多种附件：

```
工具结果
    │
    ├── ① 记忆预取（Memory Prefetch）
    │   └── pendingMemoryPrefetch → 已结算则注入
    │
    ├── ② 技能发现预取（Skill Prefetch）
    │   └── skillPrefetch → 注入发现的新技能
    │
    ├── ③ 队列命令（Queued Commands）
    │   ├── 任务通知（后台任务完成）
    │   ├── 用户排队输入
    │   └── 过滤斜杠命令（走专门处理）
    │
    ├── ④ 文件变更附件
    │   └── edited_text_file 事件
    │
    └── ⑤ MCP 工具刷新
        └── refreshTools() 更新可用工具
```

---

## 10. Continue 点完整清单

所有 `state = next; continue` 的触发条件：

| # | 触发条件 | transition.reason | 说明 |
|---|---------|-------------------|------|
| 1 | FallbackTriggeredError | — (内层 continue) | 模型降级重试 |
| 2 | Context Collapse drain | `collapse_drain_retry` | 折叠队列排空后重试 |
| 3 | Reactive Compact | `reactive_compact_retry` | 响应式压缩后重试 |
| 4 | max_output_tokens escalate | `max_output_tokens_escalate` | 8k→64k 升级重试 |
| 5 | max_output_tokens recovery | `max_output_tokens_recovery` | 注入恢复消息重试 |
| 6 | Stop hook blocking | `stop_hook_blocking` | 钩子阻塞，注入错误重试 |
| 7 | Token budget continuation | `token_budget_continuation` | token 预算用完但需要继续 |
| 8 | **正常下一轮** | `next_turn` | 工具执行完成，进入下一轮 |

---

## 11. 设计模式总结

| 模式 | 应用 | 价值 |
|------|------|------|
| **Infinite Loop + Continue** | `while(true)` + 8个 continue 点 | 统一的迭代控制流 |
| **State 单对象** | 每次迭代解构，continue 时整体赋值 | 避免 9 个独立变量 |
| **Withheld 机制** | 可恢复错误不立即 yield | 对用户透明的自动恢复 |
| **StreamingToolExecutor** | 工具流式并行执行 | 更快的工具响应 |
| **预处理管线** | 5步上下文优化 | 渐进式 token 削减 |
| **Post-sampling Hooks** | 模型响应后异步处理 | 记忆提取、遥测等不阻塞 |
| **Transition 追踪** | 每次继续记录原因 | 调试和防止死循环 |
| **依赖注入** | `deps?: QueryDeps` | 可测试性（mock API 调用） |

---

## 12. 可借鉴要点

### 12.1 Withheld 模式：优雅的错误恢复

```typescript
// 通用实现
class WithheldRecovery<T> {
  private strategies: RecoveryStrategy<T>[] = []

  add(strategy: RecoveryStrategy<T>): void {
    this.strategies.push(strategy)
  }

  async tryRecover(error: T, context: Context): Promise<RecoveryResult> {
    for (const strategy of this.strategies) {
      if (strategy.canHandle(error)) {
        const result = await strategy.recover(context)
        if (result.recovered) return result
      }
    }
    return { recovered: false, error }
  }
}
```

**适用场景**：任何需要自动重试但不想暴露中间错误的系统。

### 12.2 渐进式上下文压缩管线

```
Tool Budget → Snip → Microcompact → Collapse → Autocompact
  (最快)                                    (最慢/最贵)
```

**设计原则**：
1. 从快到慢排列
2. 每步可能让后续步骤不需要执行
3. 步骤间传递信息（如 `snipTokensFreed`）

### 12.3 State 单对象模式

```typescript
// 不推荐：多个独立变量
let messages = [], toolUseContext = {}, tracking = undefined, ...

// 推荐：单 State 对象
let state: State = { messages: [], toolUseContext: {}, tracking: undefined, ... }
// 读取
const { messages, toolUseContext } = state
// 更新
state = { ...state, messages: newMessages }
```

### 12.4 流式工具执行

```
传统模式：
  API 返回全部 tool_use → 依次执行工具 → 全部完成 → 返回结果

流式模式：
  API 流式返回 tool_use_1 → 立即开始执行
  API 流式返回 tool_use_2 → 立即开始执行
  tool_use_1 完成 → yield 结果
  tool_use_2 完成 → yield 结果
```

**延迟减少**：第一个工具的延迟 = max(API 流式延迟, 工具执行延迟)，而不是两者之和。

### 12.5 Transition 追踪防死循环

每次 continue 记录原因，恢复逻辑可以检查：
```typescript
// 防止死循环：如果上次已经是 collapse_drain_retry，不再重复
if (state.transition?.reason !== 'collapse_drain_retry') {
  // 尝试 collapse drain
}
```

### 12.6 依赖注入实现可测试性

```typescript
// 生产代码
const deps = params.deps ?? productionDeps()

// 测试代码
const mockDeps = {
  callModel: mockCallModel,
  autocompact: mockAutocompact,
  microcompact: mockMicrocompact,
  uuid: () => 'test-uuid',
}
query({ ..., deps: mockDeps })
```
