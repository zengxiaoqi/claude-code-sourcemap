# Services 模块深度源码分析

> 本文档对 `services/` 目录进行系统性分析，涵盖 API 交互、MCP 协议、上下文压缩、分析遥测、工具执行等核心服务。

## 模块概览

| 维度 | 数据 |
|------|------|
| **文件总数** | ~120 个 TypeScript/TSX 文件 |
| **核心职责** | API 通信、MCP 协议实现、上下文管理、分析遥测、工具执行编排、OAuth 认证 |
| **关键文件** | `api/claude.ts` (~3400行), `api/client.ts`, `api/withRetry.ts`, `mcp/client.ts` (~1500行), `compact/compact.ts` (~1700行), `analytics/growthbook.ts` (~1150行) |

---

## 目录

1. [模块架构图](#1-模块架构图)
2. [API 服务层](#2-api-服务层)
3. [MCP 服务层](#3-mcp-服务层)
4. [上下文压缩服务 (Compact)](#4-上下文压缩服务-compact)
5. [分析遥测服务 (Analytics)](#5-分析遥测服务-analytics)
6. [工具执行服务 (Tools)](#6-工具执行服务-tools)
7. [OAuth 认证服务](#7-oauth-认证服务)
8. [成本/额度管理服务](#8-成本额度管理服务)
9. [LSP 语言服务](#9-lsp-语言服务)
10. [插件系统 (Plugins)](#10-插件系统-plugins)
11. [其他辅助服务](#11-其他辅助服务)
12. [服务间依赖关系图](#12-服务间依赖关系图)
13. [设计优缺点分析](#13-设计优缺点分析)

---

## 1. 模块架构图

```
services/
├── api/                          # API 通信核心
│   ├── claude.ts                 # 🌟 核心查询引擎 (消息构建、流式处理、缓存)
│   ├── client.ts                 # Anthropic SDK 客户端工厂 (Bedrock/Vertex/Foundry/直连)
│   ├── withRetry.ts              # 重试引擎 (指数退避、529处理、快速模式回退)
│   ├── errors.ts                 # API 错误分类与用户消息
│   ├── errorUtils.ts             # 连接错误详情提取
│   ├── grove.ts                  # Grove 隐私设置 API
│   ├── usage.ts                  # 用量查询 API
│   ├── bootstrap.ts              # API 引导初始化
│   ├── adminRequests.ts          # 管理请求 API
│   ├── sessionIngress.ts         # 会话入口 API
│   ├── logging.ts                # API 日志记录
│   ├── metricsOptOut.ts          # 遥测退出
│   ├── referral.ts               # 推荐码 API
│   ├── overageCreditGrant.ts     # 超额信用授权
│   ├── filesApi.ts               # 文件上传 API
│   ├── promptCacheBreakDetection.ts  # 提示缓存失效检测
│   ├── firstTokenDate.ts         # 首个 token 日期
│   ├── ultrareviewQuota.ts       # 超级审查配额
│   ├── dumpPrompts.ts            # 提示转储
│   └── emptyUsage.ts             # 空用量
│
├── mcp/                          # Model Context Protocol 实现
│   ├── client.ts                 # 🌟 MCP 客户端核心 (连接、工具发现、调用)
│   ├── types.ts                  # MCP 类型定义 (配置、连接状态)
│   ├── MCPConnectionManager.tsx  # React 上下文管理器
│   ├── useManageMCPConnections.ts # MCP 连接 Hook
│   ├── auth.ts                   # MCP OAuth 认证
│   ├── config.ts                 # MCP 配置加载
│   ├── elicitationHandler.ts     # URL 引导处理器
│   ├── InProcessTransport.ts     # 进程内传输
│   ├── SdkControlTransport.ts    # SDK 控制传输
│   ├── channelAllowlist.ts       # 通道白名单
│   ├── channelNotification.ts    # 通道通知
│   ├── channelPermissions.ts     # 通道权限
│   ├── claudeai.ts               # claude.ai 代理
│   ├── headersHelper.ts          # 请求头辅助
│   ├── mcpStringUtils.ts         # 字符串工具
│   ├── normalization.ts          # 名称规范化
│   ├── oauthPort.ts              # OAuth 端口
│   ├── officialRegistry.ts       # 官方注册表
│   ├── utils.ts                  # 工具函数
│   ├── vscodeSdkMcp.ts           # VS Code SDK MCP
│   ├── xaa.ts                    # 跨应用访问
│   ├── xaaIdpLogin.ts            # XAA IdP 登录
│   └── envExpansion.ts           # 环境变量展开
│
├── compact/                      # 上下文压缩
│   ├── compact.ts                # 🌟 压缩核心逻辑 (1700+ 行)
│   ├── autoCompact.ts            # 自动压缩触发器
│   ├── prompt.ts                 # 压缩提示词模板
│   ├── grouping.ts               # 消息分组
│   ├── microCompact.ts           # 微压缩
│   ├── apiMicrocompact.ts        # API 微压缩
│   ├── sessionMemoryCompact.ts   # 会话记忆压缩
│   ├── postCompactCleanup.ts     # 压缩后清理
│   ├── compactWarningHook.ts     # 压缩警告 Hook
│   ├── compactWarningState.ts    # 压缩警告状态
│   └── timeBasedMCConfig.ts      # 基于时间的微压缩配置
│
├── analytics/                    # 分析遥测
│   ├── index.ts                  # 公共 API (logEvent、事件队列)
│   ├── sink.ts                   # 事件路由 (Datadog + 1P)
│   ├── growthbook.ts             # 🌟 Feature Flag / A-B 测试 (~1150行)
│   ├── datadog.ts                # Datadog 上报
│   ├── firstPartyEventLogger.ts  # 第一方事件日志
│   ├── firstPartyEventLoggingExporter.ts  # 1P 导出器
│   ├── config.ts                 # 分析配置
│   ├── metadata.ts               # 事件元数据提取
│   └── sinkKillswitch.ts         # Sink 熔断器
│
├── tools/                        # 工具执行编排
│   ├── toolOrchestration.ts      # 工具调度 (并发/串行分组)
│   ├── toolExecution.ts          # 🌟 单工具执行 (权限、Hook、结果处理)
│   ├── StreamingToolExecutor.ts  # 流式工具执行器
│   └── toolHooks.ts              # 工具 Hook
│
├── oauth/                        # OAuth 认证
│   ├── client.ts                 # OAuth 客户端
│   ├── index.ts                  # 入口
│   ├── crypto.ts                 # PKCE 加密
│   ├── auth-code-listener.ts     # 授权码监听
│   └── getOauthProfile.ts        # 获取 OAuth Profile
│
├── lsp/                          # Language Server Protocol
│   ├── manager.ts                # LSP 管理器入口
│   ├── LSPServerManager.ts       # 服务器管理器
│   ├── LSPServerInstance.ts      # 服务器实例
│   ├── LSPClient.ts              # LSP 客户端
│   ├── LSPDiagnosticRegistry.ts  # 诊断注册
│   ├── config.ts                 # LSP 配置
│   └── passiveFeedback.ts        # 被动反馈
│
├── plugins/                      # 插件系统
│   ├── pluginOperations.ts       # 插件操作 (安装/卸载/启用/禁用/更新)
│   ├── PluginInstallationManager.ts  # 安装管理器
│   └── pluginCliCommands.ts      # CLI 命令
│
├── SessionMemory/                # 会话记忆
│   ├── sessionMemory.ts          # 会话记忆管理
│   ├── sessionMemoryUtils.ts     # 工具函数
│   └── prompts.ts                # 提示词
│
├── autoDream/                    # 自动记忆整合
│   ├── autoDream.ts              # 后台记忆整合引擎
│   ├── config.ts                 # 配置
│   ├── consolidationLock.ts      # 整合锁
│   └── consolidationPrompt.ts    # 整合提示词
│
├── extractMemories/              # 记忆提取
│   ├── extractMemories.ts        # 记忆提取逻辑
│   └── prompts.ts                # 提示词
│
├── AgentSummary/                 # Agent 摘要
├── MagicDocs/                    # 魔法文档
├── PromptSuggestion/             # 提示建议
├── settingsSync/                 # 设置同步
├── remoteManagedSettings/        # 远程托管设置
├── teamMemorySync/               # 团队记忆同步
├── policyLimits/                 # 策略限制
├── tips/                         # 提示系统
├── toolUseSummary/               # 工具使用摘要
│
├── claudeAiLimits.ts             # Claude.ai 额度管理
├── claudeAiLimitsHook.ts         # 额度 Hook
├── rateLimitMessages.ts          # 速率限制消息
├── rateLimitMocking.ts           # 速率限制模拟
├── mockRateLimits.ts             # 模拟限制
├── tokenEstimation.ts            # Token 估算
├── internalLogging.ts            # 内部日志
├── diagnosticTracking.ts         # 诊断追踪
├── notifier.ts                   # 通知器
├── preventSleep.ts               # 防止休眠
├── mcpServerApproval.tsx         # MCP 服务器审批
├── vcr.ts                        # 录制/回放
├── voice.ts                      # 语音
├── voiceKeyterms.ts              # 语音关键词
├── voiceStreamSTT.ts             # 语音流 STT
└── awaySummary.ts                # 离开摘要
```

---

## 2. API 服务层

### 2.1 核心查询引擎 — `api/claude.ts` (~3400 行)

这是整个 CLI 与 Anthropic API 交互的核心引擎，负责：
- 消息构建与规范化
- 流式响应处理 (SSE)
- 工具调用集成
- 提示缓存管理
- 模型路由

```typescript
// api/claude.ts - 关键类型与签名
import type {
  BetaMessage,
  BetaRawMessageStreamEvent,
  BetaMessageStreamParams,
} from '@anthropic-ai/sdk/resources/beta/messages/messages.mjs'

// 主要导出函数
export function queryModelWithStreaming(...)  // 流式查询
export function getMetadata(...)              // 获取 API 元数据
export function getMaxOutputTokensForModel(model: string): number
```

**消息构建流程：**

1. **系统提示构建** — `splitSysPromptPrefix` 分割前缀与主体，利用 `logAPIPrefix` 日志前缀
2. **消息规范化** — `normalizeMessagesForAPI()` 将内部消息格式转为 SDK 参数格式
3. **工具 Schema 转换** — `toolToAPISchema()` 将内部 Tool 定义转为 API 工具格式
4. **Beta 头管理** — `getMergedBetas()` / `getModelBetas()` 合并特性标志

**流式处理架构：**

```
queryModelWithStreaming()
  → getAnthropicClient()          // 创建/复用客户端
  → buildMessageParams()          // 构建请求参数
  → client.beta.messages.stream() // 启动流
  → processStreamEvents()         // 处理 SSE 事件
    → text_delta                  // 文本增量
    → tool_use                    // 工具调用
    → thinking_delta              // 思考增量
    → message_stop                // 消息结束
```

### 2.2 客户端工厂 — `api/client.ts`

根据运行环境自动创建适配的 Anthropic SDK 客户端，支持 4 种后端：

```typescript
// api/client.ts - 多后端客户端创建
export async function getAnthropicClient({
  apiKey,
  maxRetries,
  model,
  fetchOverride,
  source,
}: { ... }): Promise<Anthropic> {

  // 1. AWS Bedrock
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK)) {
    const { AnthropicBedrock } = await import('@anthropic-ai/bedrock-sdk')
    return new AnthropicBedrock({ awsRegion, ... }) as unknown as Anthropic
  }

  // 2. Azure Foundry
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY)) {
    const { AnthropicFoundry } = await import('@anthropic-ai/foundry-sdk')
    return new AnthropicFoundry({ azureADTokenProvider, ... }) as unknown as Anthropic
  }

  // 3. Google Vertex AI
  if (isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX)) {
    const [{ AnthropicVertex }, { GoogleAuth }] = await Promise.all([...])
    return new AnthropicVertex({ region, googleAuth, ... }) as unknown as Anthropic
  }

  // 4. 直连 Anthropic API (默认)
  return new Anthropic({ apiKey, authToken, ... })
}
```

**设计亮点：**
- 每个后端的认证逻辑完全独立：AWS 凭证刷新、Azure AD Token Provider、GoogleAuth
- 自定义请求头注入 (`ANTHROPIC_CUSTOM_HEADERS`) 支持，按换行符分割的 `Name: Value` 格式
- 自动 OAuth Token 刷新（`checkAndRefreshOAuthTokenIfNeeded`）
- 请求级 UUID (`x-client-request-id`) 用于超时场景的服务端日志关联

### 2.3 重试引擎 — `api/withRetry.ts`

这是 Claude Code 最复杂的重试系统之一，实现了：

```typescript
// api/withRetry.ts - 重试核心
export async function* withRetry<T>(
  getClient: () => Promise<Anthropic>,
  operation: (client, attempt, context) => Promise<T>,
  options: RetryOptions,
): AsyncGenerator<SystemAPIErrorMessage, T>
```

**重试策略矩阵：**

| 错误类型 | 行为 | 最大重试 |
|----------|------|----------|
| 529 Overloaded | 前台源重试 (MAX_529_RETRIES=3)，后台源立即放弃 | 3 → 可配置 fallback model |
| 429 Rate Limit | 非 Claude.ai 订阅重试；Claude.ai 订阅 Enterprise 重试 | 10 (默认) |
| 401 Unauthorized | 刷新 Token 后重试 | 包含在 maxRetries |
| ECONNRESET/EPIPE | 禁用 keep-alive 并重连 | 包含在 maxRetries |
| 400 Context Overflow | 自动调整 max_tokens 并重试 | 特殊处理 |
| Fast Mode 拒绝 | 短 retry-after 保持快速模式；长 cooldown 降级标准模式 | 持续 |

**指数退避算法：**

```typescript
// api/withRetry.ts
export function getRetryDelay(
  attempt: number,
  retryAfterHeader?: string | null,
  maxDelayMs = 32000,
): number {
  if (retryAfterHeader) {
    return parseInt(retryAfterHeader, 10) * 1000  // 服务器指定优先
  }
  const baseDelay = Math.min(500 * Math.pow(2, attempt - 1), maxDelayMs) // 500 → 1000 → 2000 → ...
  const jitter = Math.random() * 0.25 * baseDelay  // 25% 抖动
  return baseDelay + jitter
}
```

**持久重试模式**（`CLAUDE_CODE_UNATTENDED_RETRY`）：
- 用于无值守会话（CI/CD）
- 最大退避 5 分钟，重置上限 6 小时
- 每 30 秒发送心跳，防止宿主标记会话空闲
- 分块 sleep，每块 yield 进度消息

**快速模式 (Fast Mode) 处理：**
- 429/529 时判断 retry-after 时间
- 短等待 (< 20s)：保持快速模式重试，利用 prompt cache
- 长等待：触发 cooldown，切换到标准速度模型
- Oage 被拒绝：永久禁用快速模式

### 2.4 错误处理 — `api/errors.ts`

```typescript
// api/errors.ts - 用户可见的错误消息
export const API_ERROR_MESSAGE_PREFIX = 'API Error'
export const REPEATED_529_ERROR_MESSAGE = '...' // 连续 529 错误提示

// 智能错误分类
export function getPromptTooLongTokenGap(response: AssistantMessage): number | undefined
// 从 "input length and `max_tokens` exceed context limit: 188059 + 20000 > 200000" 解析数值
```

---

## 3. MCP 服务层

### 3.1 架构概览

MCP (Model Context Protocol) 服务是 Claude Code 与外部工具/服务器通信的核心协议实现，文件数约 25 个，支持 7 种传输方式：

```
┌──────────────────────────────────────────────────────────────┐
│                    MCPConnectionManager (React Context)       │
│         reconnectMcpServer / toggleMcpServer                  │
├──────────────────────────────────────────────────────────────┤
│  useManageMCPConnections (Hook)                               │
│    → 管理连接生命周期、重连、toggle                              │
├──────────────────────────────────────────────────────────────┤
│  connectToServer (memoized)                                   │
│    → 创建 Transport → Client → 连接 → fetchTools/Commands    │
├──────────────────────────────────────────────────────────────┤
│  Transport 层 (7 种):                                         │
│  ┌─────────┐ ┌─────┐ ┌─────────┐ ┌────┐ ┌───────┐ ┌───┐ ┌──────────┐
│  │  stdio   │ │ sse │ │ sse-ide │ │ ws │ │ http  │ │sdk│ │claudeai  │
│  │ 子进程   │ │ SSE │ │ IDE SSE │ │ WS │ │Stream │ │进程│ │ proxy    │
│  └─────────┘ └─────┘ └─────────┘ └────┘ └───────┘ └───┘ └──────────┘
├──────────────────────────────────────────────────────────────┤
│  认证层: ClaudeAuthProvider / OAuth / XAA / SessionIngress    │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 连接管理核心 — `mcp/client.ts` (~1500 行)

这是 MCP 系统最核心的文件，负责：

**连接创建 (`connectToServer`)**：

```typescript
// mcp/client.ts - 连接创建（使用 memoize 缓存）
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: { totalServers, stdioCount, ... },
  ): Promise<MCPServerConnection> => {
    // 1. 根据 type 创建对应的 Transport
    // 2. 创建 MCP Client (name: 'claude-code')
    // 3. 设置 ListRoots 处理器 (返回 cwd)
    // 4. 带超时连接
    // 5. 注册 onerror/onclose 处理器（连接断开检测）
    // 6. 注册 cleanup（子进程终止：SIGINT → SIGTERM → SIGKILL）
  },
  getServerCacheKey,  // 缓存键：name + JSON.stringify(config)
)
```

**传输层超时管理：**

```typescript
// mcp/client.ts - 每 60s 刷新超时
export function wrapFetchWithTimeout(baseFetch: FetchLike): FetchLike {
  return async (url, init) => {
    // GET 请求跳过超时（长连接 SSE 流）
    if (method === 'GET') return baseFetch(url, init)
    
    // POST 请求使用 setTimeout（避免 AbortSignal.timeout 的内存泄漏）
    const controller = new AbortController()
    const timer = setTimeout(c => c.abort(...), 60_000, controller)
    timer.unref?.()  // 不阻止进程退出
    // ...
  }
}
```

**连接生命周期管理：**
- `onclose` 自动清除 memoize 缓存，下次调用时自动重连
- `onerror` 检测终端错误（ECONNRESET/ETIMEDOUT/EPIPE），连续 3 次触发关闭
- HTTP 会话过期检测（404 + JSON-RPC -32001），自动重连

**工具发现 (`fetchToolsForClient`)**：

```typescript
// mcp/client.ts
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => {
    const result = await client.client.request(
      { method: 'tools/list' }, ListToolsResultSchema,
    )
    return result.tools.map(tool => ({
      ...MCPTool,
      name: buildMcpToolName(client.name, tool.name),  // mcp__serverName__toolName
      mcpInfo: { serverName: client.name, toolName: tool.name },
      call: async (args, context, ...) => {
        // 1. ensureConnectedClient() — 确保连接有效
        // 2. callMCPToolWithUrlElicitationRetry() — 带重试的工具调用
        // 3. processMCPResult() — 结果处理（截断/持久化）
      }
    }))
  },
  client => client.name,  // 缓存键
  20,                     // LRU 大小
)
```

**批量连接 (`getMcpToolsCommandsAndResources`)**：

```typescript
// mcp/client.ts - 分组并行连接
const localServers = configEntries.filter(isLocalMcpServer)    // stdio/sdk
const remoteServers = configEntries.filter(!isLocalMcpServer)  // sse/http/ws

await Promise.all([
  processBatched(localServers, 3, processServer),   // 本地：低并发
  processBatched(remoteServers, 20, processServer),  // 远程：高并发
])
```

### 3.3 类型系统 — `mcp/types.ts`

```typescript
// mcp/types.ts - 连接状态联合类型
export type MCPServerConnection =
  | ConnectedMCPServer    // 已连接：有 client, capabilities, cleanup
  | FailedMCPServer       // 失败：有 error 描述
  | NeedsAuthMCPServer    // 需认证：等待 OAuth
  | PendingMCPServer      // 等待中：重连中
  | DisabledMCPServer     // 已禁用

// 配置类型（7 种传输方式的联合类型）
export type McpServerConfig =
  | McpStdioServerConfig      // 子进程
  | McpSSEServerConfig        // SSE 远程
  | McpSSEIDEServerConfig     // IDE SSE
  | McpWebSocketIDEServerConfig  // IDE WebSocket
  | McpHTTPServerConfig       // Streamable HTTP
  | McpWebSocketServerConfig  // WebSocket
  | McpSdkServerConfig        // 进程内 SDK
  | McpClaudeAIProxyServerConfig  // claude.ai 代理
```

**认证缓存：**
- `isMcpAuthCached()` — 15 分钟 TTL 文件缓存，避免反复探测需要认证的服务器
- `createClaudeAiProxyFetch()` — 代理 fetch 包装器，自动附加 Bearer Token + 401 重试

---

## 4. 上下文压缩服务 (Compact)

### 4.1 架构

上下文压缩是 Claude Code 管理长对话的核心机制，在 token 使用接近上下文窗口限制时自动触发。

```
┌─────────────────────────────────────────────────────┐
│ autoCompact.ts — 自动压缩触发器                       │
│   getAutoCompactThreshold() → 计算触发阈值           │
│   shouldAutoCompact() → 判断是否需要压缩              │
├─────────────────────────────────────────────────────┤
│ compact.ts — 压缩核心 (1700+ 行)                     │
│   compactConversation() → 主入口                     │
│     1. stripImagesFromMessages() — 移除图片           │
│     2. stripReinjectedAttachments() — 移除重注入附件  │
│     3. 消息分组 (groupMessagesByApiRound)             │
│     4. 构建压缩请求 (getCompactPrompt)                │
│     5. 调用 API 获取摘要                              │
│     6. 后压缩清理 (postCompactCleanup)                │
├─────────────────────────────────────────────────────┤
│ prompt.ts — 压缩提示词                                │
│   BASE_COMPACT_PROMPT — 全量压缩                      │
│   PARTIAL_COMPACT_PROMPT — 部分压缩                   │
│   PARTIAL_COMPACT_UP_TO_PROMPT — 前缀压缩             │
├─────────────────────────────────────────────────────┤
│ 微压缩层:                                             │
│   microCompact.ts — 消息级微压缩                      │
│   apiMicrocompact.ts — API 级微压缩                  │
│   sessionMemoryCompact.ts — 会话记忆压缩              │
│   timeBasedMCConfig.ts — 时间基微压缩配置             │
└─────────────────────────────────────────────────────┘
```

### 4.2 压缩策略

**自动压缩阈值：**

```typescript
// compact/autoCompact.ts
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000

export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

**三种压缩模式：**

| 模式 | 提示词 | 场景 |
|------|--------|------|
| 全量压缩 | `BASE_COMPACT_PROMPT` | 整个对话的摘要 |
| 部分压缩 (from) | `PARTIAL_COMPACT_PROMPT` | 仅压缩近期消息 |
| 部分压缩 (up_to) | `PARTIAL_COMPACT_UP_TO_PROMPT` | 压缩前缀，保留尾部 |

**压缩 Prompt 结构 (9 个固定段)：**

```
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Current Work (up_to 模式为 Work Completed)
9. Optional Next Step (up_to 模式为 Context for Continuing Work)
```

**Prompt-to-Long 重试 (`truncateHeadForPTLRetry`)**：
- 当压缩请求本身超出上下文限制时
- 按消息分组丢弃最旧的 API 轮次
- 每次丢弃覆盖 token gap 的最小组数
- 最多 3 次重试 (`MAX_PTL_RETRIES`)

**后压缩清理：**
- 恢复最近使用的文件（最多 5 个，每文件 5000 token）
- 重新注入 Skills（总预算 25000 token，每 skill 5000 token）
- 恢复计划文件、代理列表

### 4.3 关键代码：压缩执行

```typescript
// compact/compact.ts - 核心压缩流程 (简化)
export async function compactConversation({
  messages,
  model,
  customInstructions,
  toolUseContext,
  partialDirection,
}: CompactParams): Promise<CompactionResult> {
  // 1. 预处理
  let processedMessages = stripImagesFromMessages(messages)
  processedMessages = stripReinjectedAttachments(processedMessages)

  // 2. 分组 & 选择要压缩的范围
  const groups = groupMessagesByApiRound(processedMessages)

  // 3. 构建请求
  const compactPrompt = partialDirection
    ? getPartialCompactPrompt(customInstructions, partialDirection)
    : getCompactPrompt(customInstructions)

  // 4. 使用 forked agent 执行压缩（利用 prompt cache）
  const summary = await runForkedAgent({
    model: getSmallFastModel(),  // 使用 Haiku 降低成本
    maxTokens: COMPACT_MAX_OUTPUT_TOKENS,
    messages: messagesToCompact,
    systemPrompt: asSystemPrompt(compactPrompt),
  })

  // 5. 后处理
  const formattedSummary = formatCompactSummary(summary) // 移除 <analysis> 块
  return {
    boundaryMarker: createCompactBoundaryMessage(...),
    summary: formattedSummary,
  }
}
```

---

## 5. 分析遥测服务 (Analytics)

### 5.1 架构

```
logEvent(eventName, metadata)          // 公共 API
  → 事件队列 (attachSink 前)
  → AnalyticsSink.logEvent()
    ├── sink.ts — 路由
    │   ├── shouldSampleEvent() — 采样控制
    │   ├── trackDatadogEvent() — Datadog (stripProtoFields)
    │   └── logEventTo1P() — 第一方事件日志 (保留 PII)
    │
    └── growthbook.ts — Feature Flag
        ├── GrowthBook SDK 初始化
        ├── getFeatureValue_CACHED_MAY_BE_STALE()
        └── checkStatsigFeatureGate_CACHED_MAY_BE_STALE()
```

### 5.2 公共 API — `analytics/index.ts`

```typescript
// analytics/index.ts - 类型安全的分析 API

// 强制验证元数据不包含敏感信息的标记类型
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never

// PII 标记类型（仅 1P 导出器可见）
export type AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED = never

// 剥离 _PROTO_* 字段（非 1P 后端调用前）
export function stripProtoFields<V>(metadata: Record<string, V>): Record<string, V>

// 主 API
export function logEvent(eventName: string, metadata: LogEventMetadata): void
export function logEventAsync(eventName: string, metadata: LogEventMetadata): Promise<void>
```

### 5.3 Feature Flag — `analytics/growthbook.ts` (~1150 行)

```typescript
// analytics/growthbook.ts
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
}
```

- 使用 GrowthBook SDK 管理特性标志
- 缓存策略：内存缓存 + 持久化到全局配置
- `getFeatureValue_CACHED_MAY_BE_STALE()` — 允许返回过期值以避免启动阻塞
- 支持按用户属性（订阅类型、平台、组织等）的定向规则

### 5.4 事件路由 — `analytics/sink.ts`

```typescript
// sink.ts - 双后端路由
function logEventImpl(eventName: string, metadata: LogEventMetadata): void {
  const sampleResult = shouldSampleEvent(eventName)  // 采样
  if (sampleResult === 0) return

  if (shouldTrackDatadog()) {
    trackDatadogEvent(eventName, stripProtoFields(metadata))  // 剥离 PII
  }
  logEventTo1P(eventName, metadata)  // 保留完整数据
}
```

---

## 6. 工具执行服务 (Tools)

### 6.1 三层架构

```
toolOrchestration.ts        toolExecution.ts             StreamingToolExecutor.ts
  ┌───────────────┐         ┌──────────────────┐        ┌───────────────────────┐
  │ runTools()    │         │ runToolUse()     │        │ StreamingToolExecutor │
  │ 分组调度       │   ──→   │ 单工具执行        │   ──→  │ 流式并发执行           │
  │ 并发/串行     │         │ 权限检查          │        │ 结果缓冲有序输出       │
  └───────────────┘         │ Hook 执行        │        └───────────────────────┘
                            │ 结果处理          │
                            └──────────────────┘
```

### 6.2 工具调度 — `tools/toolOrchestration.ts`

```typescript
// tools/toolOrchestration.ts
export async function* runTools(
  toolUseMessages: ToolUseBlock[],
  assistantMessages: AssistantMessage[],
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
): AsyncGenerator<MessageUpdate, void> {
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(...)) {
    if (isConcurrencySafe) {
      // 只读工具并发执行
      for await (const update of runToolsConcurrently(blocks, ...)) {
        yield update
      }
    } else {
      // 非只读工具串行执行
      for await (const update of runToolsSerially(blocks, ...)) {
        yield update
      }
    }
  }
}
```

**分区策略：**
- 通过 `tool.isConcurrencySafe()` / `tool.isReadOnly()` 判断
- 只读工具（FileRead、Grep 等）可并发执行
- 写操作工具（Bash、Edit、Write 等）串行执行
- 最大并发数：`CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY`（默认 10）

### 6.3 流式工具执行器 — `tools/StreamingToolExecutor.ts`

```typescript
// StreamingToolExecutor.ts - 核心设计
export class StreamingToolExecutor {
  // 工具按到达顺序缓冲，结果按完成顺序收集
  // 但输出保持原始顺序（FIFO）
  private tools: TrackedTool[] = []
  // Bash 工具错误时，兄弟进程通过 siblingAbortController 立即终止
  private siblingAbortController: AbortController
}
```

**状态机：**
```
queued → executing → completed → yielded
                  ↘ error
```

---

## 7. OAuth 认证服务

### 7.1 架构

```
oauth/
├── client.ts         # PKCE 授权码流
├── crypto.ts         # code_verifier / code_challenge 生成
├── auth-code-listener.ts  # 本地 HTTP 服务器监听回调
├── getOauthProfile.ts     # 获取用户 Profile
├── index.ts          # 入口
└── types.ts          # Token/Profile 类型定义
```

### 7.2 核心流程

```typescript
// oauth/client.ts
export function buildAuthUrl({
  codeChallenge,  // PKCE challenge
  state,          // CSRF 防护
  port,           // 回调端口
  isManual,       // 手动模式（无浏览器）
  loginWithClaudeAi,
  inferenceOnly,
  orgUUID,
  loginHint,
  loginMethod,
}: { ... }): string  // 返回授权 URL
```

- 支持 `ALL_OAUTH_SCOPES` 和 `CLAUDE_AI_OAUTH_SCOPES` 两种范围
- PKCE (Proof Key for Code Exchange) 保护
- 本地 HTTP 服务器监听回调或手动输入授权码

---

## 8. 成本/额度管理服务

### 8.1 `claudeAiLimits.ts` — Claude.ai 订阅额度

```typescript
// claudeAiLimits.ts - 额度类型
type RateLimitType =
  | 'five_hour'           // 5 小时窗口
  | 'seven_day'           // 7 天窗口
  | 'seven_day_opus'      // 7 天 Opus 窗口
  | 'seven_day_sonnet'    // 7 天 Sonnet 窗口
  | 'overage'             // 超额

type QuotaStatus = 'allowed' | 'allowed_warning' | 'rejected'

// 早期预警配置
const EARLY_WARNING_CONFIGS: EarlyWarningConfig[] = [
  { rateLimitType: 'five_hour', thresholds: [{ utilization: 0.9, timePct: 0.72 }] },
  // ...
]
```

### 8.2 `api/usage.ts` — 用量查询

```typescript
// api/usage.ts
export async function fetchUtilization(): Promise<Utilization | null> {
  // 返回各窗口的利用率 (0-100%) 和重置时间
  type Utilization = {
    five_hour?: RateLimit | null
    seven_day?: RateLimit | null
    seven_day_opus?: RateLimit | null
    extra_usage?: ExtraUsage | null
  }
}
```

---

## 9. LSP 语言服务

### 9.1 架构

```
lsp/
├── manager.ts             # 单例管理器入口
├── LSPServerManager.ts    # 服务器管理器（创建/销毁实例）
├── LSPServerInstance.ts   # 单个 LSP 服务器实例
├── LSPClient.ts           # LSP 客户端
├── LSPDiagnosticRegistry.ts  # 诊断注册
├── config.ts              # LSP 配置
└── passiveFeedback.ts     # 被动反馈（诊断通知）
```

- 使用 `InitializationState` 管理初始化生命周期：`not-started` → `pending` → `success` / `failed`
- Generation counter 防止过时的初始化 Promise 更新状态
- 提供全局单例访问 `getLspServerManager()`

---

## 10. 插件系统 (Plugins)

### 10.1 架构

```typescript
// plugins/pluginOperations.ts - 纯库函数
// 安装、卸载、启用、禁用、更新操作
// 不调用 process.exit()，不直接写 console
// 返回结果对象 { success, message }
```

**插件生命周期：**
1. 解析插件标识符 (`parsePluginIdentifier`)
2. 从 Marketplace 获取插件信息 (`getPluginById`)
3. 版本解析与缓存 (`calculatePluginVersion`)
4. 安装到磁盘 (`installResolvedPlugin`)
5. 缓存管理 (`cachePlugin`, `copyPluginToVersionedCache`)
6. 依赖解析 (`findReverseDependents`)
7. 策略检查 (`isPluginBlockedByPolicy`)

---

## 11. 其他辅助服务

### 11.1 会话记忆 (`SessionMemory/`)
- 自动维护会话 Markdown 笔记文件
- 后台 forked agent 提取关键信息
- 基于工具调用次数阈值触发更新

### 11.2 自动记忆整合 (`autoDream/`)
- 后台记忆整合引擎（"做梦"）
- 时间门控：距上次整合 >= minHours
- 会话门控：新会话数 >= minSessions
- 锁机制：`tryAcquireConsolidationLock` 防止并发整合

### 11.3 记忆提取 (`extractMemories/`)
- 从对话中提取值得记忆的信息
- 使用 forked agent 执行

### 11.4 提示建议 (`PromptSuggestion/`)
- 为用户生成下一步建议
- 包含投机预取 (`speculation.ts`)

### 11.5 Token 估算 (`tokenEstimation.ts`)
- 粗略 token 计数，用于压缩前的快速判断

### 11.6 设置同步 (`settingsSync/`, `remoteManagedSettings/`)
- 远程设置同步与缓存
- 企业级策略管理

### 11.7 团队记忆同步 (`teamMemorySync/`)
- 团队级记忆共享
- 密钥扫描器 (`secretScanner.ts`) 防止泄露

### 11.8 通知 (`notifier.ts`)
- 系统通知发送

### 11.9 防休眠 (`preventSleep.ts`)
- 防止长时间操作期间系统休眠

### 11.10 VCR (`vcr.ts`)
- 录制/回放 API 响应，用于测试

### 11.11 语音服务 (`voice.ts`, `voiceKeyterms.ts`, `voiceStreamSTT.ts`)
- TTS/STT 语音功能

---

## 12. 服务间依赖关系图

```
                    ┌──────────────────────┐
                    │  analytics/          │
                    │  (logEvent, GrowthBook)│
                    │  ← 所有服务都依赖      │
                    └──────────┬───────────┘
                               │
    ┌──────────────┬───────────┼───────────┬──────────────┐
    │              │           │           │              │
    ▼              ▼           ▼           ▼              ▼
┌────────┐  ┌──────────┐ ┌────────┐ ┌─────────┐  ┌──────────┐
│ api/   │  │ compact/ │ │ tools/ │ │   mcp/  │  │ oauth/   │
│ claude │←─│ compact  │ │toolExec│ │ client  │  │ client   │
│ client │  │ autoComp │ │orchest │ │ types   │  │ crypto   │
│ retry  │  │ prompt   │ │stream  │ │ config  │  │ listener │
└───┬────┘  └────┬─────┘ └───┬────┘ └───┬─────┘  └──────────┘
    │            │            │           │
    │            └────────────┤           │
    │                         │           │
    │  ┌──────────────────────┤           │
    │  │                      │           │
    ▼  ▼                      ▼           ▼
┌───────────────────────────────────────────────────┐
│                 共享工具层 (utils/)                  │
│  auth.js │ http.js │ messages.js │ tokens.js       │
│  config.js │ hooks.js │ forkedAgent.js │ ...       │
└───────────────────────────────────────────────────┘

关键依赖路径：
  compact/ ──→ api/claude.ts ──→ api/client.ts ──→ Anthropic SDK
  compact/ ──→ utils/forkedAgent.ts ──→ api/claude.ts (压缩用 Haiku)
  tools/  ──→ mcp/client.ts ──→ MCP SDK
  tools/  ──→ api/withRetry.ts (MCP 工具调用重试)
  analytics/ ──→ growthbook.ts (Feature Flag 影响 API 行为)
```

---

## 13. 设计优缺点分析

### 13.1 优点

**1. 多后端透明支持**
- `client.ts` 通过工厂模式，统一了 Bedrock/Vertex/Foundry/直连四种后端
- 每个后端的认证逻辑完全隔离，添加新后端只需增加一个 `if` 分支

**2. 精密的重试策略**
- `withRetry.ts` 针对不同错误类型（529/429/401/连接错误）制定了差异化策略
- 前台/后台源区分，避免了后台查询在容量洪峰期间的放大效应
- 持久重试模式专为无值守场景设计，带心跳保活

**3. MCP 协议的全面实现**
- 7 种传输方式完整覆盖（stdio/sse/http/ws/sdk/ide/proxy）
- 认证缓存（15min TTL）避免重复探测
- 连接生命周期完整管理（创建→监控→错误检测→优雅关闭→自动重连）

**4. 上下文压缩的分层设计**
- 全量/部分/前缀三种压缩模式适应不同场景
- Prompt-to-Long 兜底机制确保用户不会被"卡住"
- 后压缩清理恢复关键上下文（文件、技能、计划）

**5. 遥测的隐私保护**
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 强制代码审查者确认不含敏感数据
- `_PROTO_*` 前缀的 PII 字段仅 1P 管道可见
- 采样率控制减少遥测流量

### 13.2 缺点与改进空间

**1. 单文件过度膨胀**
- `api/claude.ts` (~3400 行)、`compact/compact.ts` (~1700 行)、`mcp/client.ts` (~1500 行) 严重超出单文件合理范围
- 建议按职责拆分：消息构建、流式处理、缓存管理分离为独立模块

**2. memoize 缓存策略复杂**
- `mcp/client.ts` 中 `connectToServer` 使用 lodash memoize，但缓存失效依赖 `onclose` 回调手动清除
- 多处缓存（connection/fetchTools/fetchCommands/fetchResources）需要同步清除
- 建议引入统一的缓存管理器，自动协调失效

**3. 错误处理的重复模式**
- `withRetry.ts` 中大量 `if (error instanceof APIError && error.status === xxx)` 模式
- 建议抽象为错误分类器（ErrorClassifier），按策略模式处理

**4. 配置散落**
- 大量通过 `process.env.XXX` 直接读取的环境变量（如 `CLAUDE_CODE_USE_BEDROCK`、`CLAUDE_CODE_MAX_RETRIES`、`MCP_TOOL_TIMEOUT`）
- 缺乏集中式配置验证和文档
- 建议引入配置 schema（如 zod），启动时统一验证

**5. 强耦合 React 上下文**
- `MCPConnectionManager.tsx` 将 MCP 连接管理与 React Context 绑定
- 注释中已承认 "We may be able to get rid of this context by putting these function on app state"
- 建议 MCP 核心逻辑与 UI 框架解耦

**6. 类型安全的名义类型**
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never` 是通过类型断言 `as` 实现的名义类型
- 实际上不提供编译时保护，任何字符串都可以通过 `as` 断言绕过
- 建议使用 branded type 或编译时 lint 规则增强

**7. 工具执行器的复杂性**
- `StreamingToolExecutor.ts` 同时管理状态机、并发控制、错误传播
- 与 `toolOrchestration.ts` 的分区逻辑存在职责重叠
- 建议统一为一个清晰的工具执行管道

---

## 附录：文件分类统计

| 分类 | 文件数 | 代表文件 |
|------|--------|----------|
| API 通信 | 19 | claude.ts, client.ts, withRetry.ts |
| MCP 协议 | 25 | client.ts, types.ts, MCPConnectionManager.tsx |
| 上下文压缩 | 10 | compact.ts, autoCompact.ts, prompt.ts |
| 分析遥测 | 8 | index.ts, sink.ts, growthbook.ts |
| 工具执行 | 4 | toolOrchestration.ts, toolExecution.ts, StreamingToolExecutor.ts |
| OAuth 认证 | 5 | client.ts, crypto.ts |
| LSP 服务 | 7 | manager.ts, LSPServerManager.ts |
| 插件系统 | 3 | pluginOperations.ts |
| 会话记忆 | 6 | sessionMemory.ts, autoDream.ts, extractMemories.ts |
| 设置/同步 | 8 | settingsSync/, remoteManagedSettings/, teamMemorySync/ |
| 额度/限制 | 5 | claudeAiLimits.ts, rateLimitMessages.ts |
| 辅助服务 | 15+ | notifier.ts, preventSleep.ts, vcr.ts, voice*.ts, tips/ |
