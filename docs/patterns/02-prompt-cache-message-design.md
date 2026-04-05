# 设计模式：Prompt Cache 友好的消息设计

## 1. 模式概述

**问题**：多 Agent 并行时，每个子 Agent 都需要发送 API 请求。如果消息结构不一致，LLM Provider 的 Prompt Cache 无法命中，导致大量重复 token 的费用和延迟。

**解决方案**：通过精心的消息结构设计，确保所有子 Agent 的 API 请求共享相同的前缀，最大化 Prompt Cache 命中率。

**核心原理**：LLM Provider（Anthropic、OpenAI）的 Prompt Cache 基于**字节级前缀匹配**——只要多个请求的消息前缀完全一致，就可以复用缓存。

---

## 2. 源码实现分析

### 2.1 Fork 路径的消息构建

> 文件：`tools/AgentTool/forkSubagent.ts`

这是整个 Cache 策略的核心。`buildForkedMessages()` 函数展示了如何为子 Agent 构建字节级相同的消息前缀：

```typescript
/**
 * Build the forked conversation messages for the child agent.
 *
 * For prompt cache sharing, all fork children must produce byte-identical
 * API request prefixes. This function:
 * 1. Keeps the full parent assistant message (all tool_use blocks, thinking, text)
 * 2. Builds a single user message with tool_results for every tool_use block
 *    using an identical placeholder, then appends a per-child directive text block
 *
 * Result: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
 * Only the final text block differs per child, maximizing cache hits.
 */
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆 assistant 消息（保留所有 content blocks）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...assistantMessage.message.content],
    },
  }

  // 2. 收集所有 tool_use blocks
  const toolUseBlocks = assistantMessage.message.content.filter(
    (block): block is BetaToolUseBlock => block.type === 'tool_use',
  )

  // 3. 所有 tool_result 使用完全相同的占位符文本
  const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
  
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text' as const, text: FORK_PLACEHOLDER_RESULT }],
  }))

  // 4. 构建单条 user 消息：占位符 results + 唯一的 directive
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text' as const, text: buildChildMessage(directive) },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**关键设计点**：

```
父对话：  [...历史消息, assistant(tool_use_1, tool_use_2)]
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                                       ▼
子 Agent A：  [...历史消息, assistant(tool_use_1, tool_use_2),
               user(placeholder_result_1, placeholder_result_2, "指令A")]
                    ▲                                       ▲
                    │  字节级相同的前缀                       │
子 Agent B：  [...历史消息, assistant(tool_use_1, tool_use_2),
               user(placeholder_result_1, placeholder_result_2, "指令B")]
                                                         ▲
                                              只有最后一个 text block 不同
```

### 2.2 系统 Prompt 的渲染与传递

> 文件：`tools/AgentTool/forkSubagent.ts`

Fork Agent 定义中有一个关键设计——**不重新渲染系统 Prompt**：

```typescript
/**
 * Not registered in builtInAgents — used only when `!subagent_type` and the
 * experiment is active. `tools: ['*']` with `useExactTools` means the fork
 * child receives the parent's exact tool pool (for cache-identical API
 * prefixes). `permissionMode: 'bubble'` surfaces permission prompts to the
 * parent terminal. `model: 'inherit'` keeps the parent's model for context
 * length parity.
 *
 * The getSystemPrompt here is unused: the fork path passes
 * `override.systemPrompt` with the parent's already-rendered system prompt
 * bytes, threaded via `toolUseContext.renderedSystemPrompt`. Reconstructing
 * by re-calling getSystemPrompt() can diverge (GrowthBook cold→warm) and
 * bust the prompt cache; threading the rendered bytes is byte-exact.
 */
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  tools: ['*'],              // 使用父级的完整工具池
  maxTurns: 200,
  model: 'inherit',          // 继承父级的模型
  permissionMode: 'bubble',  // 权限冒泡到父终端
  getSystemPrompt: () => '', // 未使用！直接传递已渲染的字节
}
```

**为什么不能重新渲染？**
- GrowthBook Feature Flags 可能在两次调用间从 cold 变为 warm（缓存状态不同）
- 任何微小的差异（空格、顺序、Flag 值）都会导致前缀不匹配
- 直接传递已渲染的字节是**字节级精确**的

### 2.3 工具定义的缓存一致性

> 文件：`forkSubagent.ts`

```typescript
// tools: ['*'] + useExactTools 意味着 fork 子 Agent 接收父级的完整工具池
// 这确保了 API 请求中的工具定义部分也是字节级相同的
```

`tools: ['*']` 配合 `useExactTools` 策略，确保所有 fork 子 Agent 的工具定义完全一致。

### 2.4 占位符模式

```typescript
/** Placeholder text used for all tool_result blocks in the fork prefix.
 * Must be identical across all fork children for prompt cache sharing. */
const FORK_PLACEHOLDER_RESULT = 'Fork started — processing in background'
```

所有子 Agent 的 tool_result 使用**完全相同的占位符文本**。这样做：
1. 确保前缀的字节级一致性
2. 子 Agent 看到占位符后知道自己在 fork 模式
3. 占位符文本不包含任何可能变化的信息

### 2.5 防止递归 Fork

```typescript
/**
 * Guard against recursive forking. Fork children keep the Agent tool in their
 * tool pool for cache-identical tool definitions, so we reject fork attempts
 * at call time by detecting the fork boilerplate tag in conversation history.
 */
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

注意：Fork 子 Agent 保留了 Agent 工具（为了工具定义一致），但在调用时会被拒绝。

---

## 3. 设计要点

### 3.1 Prefix Caching 原理

```
API 请求：[system_prompt][messages...][tools]
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │     Prompt Cache 匹配引擎      │
                    │  逐字节比较前缀是否一致          │
                    └───────────────────────────────┘
                          │               │
                     前缀匹配          前缀不同
                          │               │
                    ▼                    ▼
              缓存命中               缓存未命中
           (0 延迟, 节省费用)     (全量处理, 全额费用)
```

### 3.2 消息不可变性

已发送的消息不应该被修改。Fork 机制通过**克隆**而非**修改**来构建子 Agent 消息：

```typescript
const fullAssistantMessage: AssistantMessage = {
  ...assistantMessage,                    // 浅拷贝
  uuid: randomUUID(),                     // 新 UUID
  message: {
    ...assistantMessage.message,
    content: [...assistantMessage.message.content],  // 新数组
  },
}
```

### 3.3 延迟渲染策略

系统 Prompt 不在定义时渲染，而是在实际发送时才渲染（`renderedSystemPrompt`）。Fork 路径直接传递已渲染的字节，跳过重新渲染。

### 3.4 工具定义一致性

`tools: ['*']` 确保子 Agent 使用父级的完整工具池，API 请求中的工具定义部分完全一致。

---

## 4. 可落地方案

### 4.1 消息结构设计规范

```typescript
// 消息构建器：确保缓存友好的消息结构
export class CacheAwareMessageBuilder {
  private systemPrompt?: string
  private tools?: ToolDefinition[]

  /**
   * 设置缓存不变的系统提示（所有子 Agent 共享）
   */
  setCacheablePrefix(systemPrompt: string, tools: ToolDefinition[]): void {
    this.systemPrompt = systemPrompt
    this.tools = tools
  }

  /**
   * 为子 Agent 构建消息（只有最后的 directive 不同）
   */
  buildChildMessages(
    parentHistory: Message[],
    parentAssistantMsg: AssistantMessage,
    directive: string,
  ): Message[] {
    // 1. 保持父对话历史不变
    // 2. 保持 assistant 消息不变
    // 3. 用占位符填充 tool_results
    // 4. 只有最后的 directive 不同
    
    const placeholder = 'Processing in background'
    const toolResults = parentAssistantMsg.toolUses.map(tu => ({
      type: 'tool_result',
      tool_use_id: tu.id,
      content: [{ type: 'text', text: placeholder }],
    }))

    return [
      ...parentHistory,
      { ...parentAssistantMsg },  // 克隆
      {
        type: 'user',
        content: [
          ...toolResults,
          { type: 'text', text: directive },
        ],
      },
    ]
  }
}
```

### 4.2 Cache 效果度量

```typescript
export class CacheMetrics {
  private hits = 0
  private misses = 0

  recordHit(): void { this.hits++ }
  recordMiss(): void { this.misses++ }

  get hitRate(): number {
    const total = this.hits + this.misses
    return total === 0 ? 0 : this.hits / total
  }

  get savings(): string {
    // Anthropic: cache 读取是正常价格的 10%
    // OpenAI: cache 读取是正常价格的 50%
    return `${(this.hitRate * 90).toFixed(1)}% cost saved (Anthropic)`
  }
}
```

### 4.3 不同 LLM Provider 适配

```typescript
export interface CacheStrategy {
  markCacheable(content: string): string  // 添加 cache_control 标记
  detectHit(response: APIResponse): boolean
}

// Anthropic 实现
export class AnthropicCacheStrategy implements CacheStrategy {
  markCacheable(content: string): string {
    // Anthropic 使用 cache_control: { type: "ephemeral" }
    return content  // 通过 API 参数传递，不在内容中标记
  }
  
  detectHit(response: any): boolean {
    return response.usage?.cache_read_input_tokens > 0
  }
}

// OpenAI 实现
export class OpenAICacheStrategy implements CacheStrategy {
  markCacheable(content: string): string {
    // OpenAI 自动缓存，无需标记
    return content
  }

  detectHit(response: any): boolean {
    return response.usage?.prompt_tokens_details?.cached_tokens > 0
  }
}
```

---

## 5. 总结

Claude Code 的 Prompt Cache 策略核心是**字节级前缀一致性**：

1. **Fork 消息构建**：所有子 Agent 共享历史 + assistant 消息 + 占位符 results，只有最后的 directive 不同
2. **系统 Prompt 直传**：不重新渲染，避免 Feature Flag 变化导致的前缀偏差
3. **工具定义一致**：`tools: ['*']` 确保工具池相同
4. **占位符模式**：统一的 `FORK_PLACEHOLDER_RESULT` 文本
5. **递归防护**：`isInForkChild()` 检测防止嵌套 fork

这套设计在多 Agent 并行场景下可以节省 **60-90%** 的 API 成本（取决于对话历史长度），是构建生产级多 Agent 系统的关键优化。
