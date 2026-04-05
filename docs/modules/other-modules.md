# Claude Code 次要模块合集 — 深度源码分析

> 本文档覆盖 claude-code-sourcemap 项目中除 core、services、tools、hooks、utils 等核心模块外的 **27 个模块**，逐一分析其功能、架构与设计要点。

---

## 1. skills/ — 技能发现、加载与执行

### 功能概述
技能系统是 Claude Code 的可扩展命令机制。用户可在 `.claude/skills/` 目录放置 Markdown 文件定义自定义命令，系统也内置了一批 bundled skills。技能本质上是带 frontmatter 元数据的 Markdown 提示词，被解析后注册为 slash command。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `loadSkillsDir.ts` | ~700 | 核心加载器：扫描目录、解析 frontmatter、注册命令 |
| `bundledSkills.ts` | 220 | 内置技能注册机制 |
| `mcpSkillBuilders.ts` | 44 | MCP 技能构建器的注册表（打破循环依赖） |
| `bundled/index.ts` | — | 内置技能定义入口 |

### 核心代码片段

**技能加载器（loadSkillsDir.ts）** 负责从多个来源发现并加载技能：

```typescript
export type LoadedFrom =
  | 'commands_DEPRECATED'
  | 'skills'
  | 'plugin'
  | 'managed'
  | 'bundled'
  | 'mcp'

export function getSkillsPath(
  source: SettingSource | 'plugin',
  dir: 'skills' | 'commands',
): string
```

加载流程：
1. 遍历项目目录向上至 home 目录，收集 `.claude/skills/` 和 `.claude/commands/` 下的 Markdown 文件
2. 解析 frontmatter（参数定义、允许工具列表、模型覆盖等）
3. 为每个技能创建 `Command` 对象，注册到命令系统
4. 支持参数替换（`$ARGUMENTS`）和 shell 命令执行

**BundledSkillDefinition** 定义了内置技能的结构：

```typescript
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  allowedTools?: string[]
  model?: string
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>
  getPromptForCommand: (args: string, context: ToolUseContext) => Promise<ContentBlockParam[]>
}
```

**MCP Skill Builders** 使用注册表模式打破循环依赖：

```typescript
let builders: MCPSkillBuilders | null = null
export function registerMCPSkillBuilders(b: MCPSkillBuilders): void { builders = b }
export function getMCPSkillBuilders(): MCPSkillBuilders { ... }
```

### 设计要点
- **多来源合并**：技能可来自项目目录、用户全局目录、插件、MCP 服务器或内置定义，后加载的覆盖先加载的
- **Frontmatter 解析**：支持复杂的前置元数据，包括参数定义、shell 执行、工具白名单
- **惰性提取**：内置技能的 reference files 在首次调用时才提取到磁盘
- **循环依赖处理**：MCP 技能通过注册表间接引用加载函数，避免 `client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts` 的循环

---

## 2. plugins/ — 插件注册与生命周期

### 功能概述
插件系统允许用户通过 `/plugin` UI 启用/禁用功能模块。内置插件（built-in）与市场插件（marketplace）分离，内置插件随 CLI 发布，用户可控制启用状态。

### 关键文件
| 文件 | 职责 |
|------|------|
| `builtinPlugins.ts` | 内置插件注册表、启用/禁用逻辑 |
| `bundled/index.ts` | 内置插件打包入口 |

### 核心代码片段

```typescript
const BUILTIN_PLUGINS: Map<string, BuiltinPluginDefinition> = new Map()
export const BUILTIN_MARKETPLACE_NAME = 'builtin'

export function registerBuiltinPlugin(definition: BuiltinPluginDefinition): void {
  BUILTIN_PLUGINS.set(definition.name, definition)
}

export function getBuiltinPlugins(): { enabled: LoadedPlugin[]; disabled: LoadedPlugin[] } {
  const settings = getSettings_DEPRECATED()
  for (const [name, definition] of BUILTIN_PLUGINS) {
    const isEnabled = userSetting !== undefined
      ? userSetting === true
      : (definition.defaultEnabled ?? true)
  }
}
```

### 设计要点
- **Plugin ID 格式**：`{name}@builtin` vs `{name}@{marketplace}`，统一区分来源
- **用户控制**：通过 `settings.json` 中的 `enabledPlugins` 持久化启用状态
- **可用性检查**：`isAvailable()` 钩子让插件在运行时决定是否展示（如依赖特定 feature flag）
- **与技能系统的区别**：插件是更重的封装，可同时包含 skills + hooks + MCP servers

---

## 3. bridge/ — 双向通信协议

### 功能概述
Bridge 是 Claude Code 的远程控制基础设施。它允许外部客户端（如 claude.ai 网页端）通过 Environments API 创建工作会话，建立双向消息通道，实现远程交互、权限审批和会话管理。这是 CCR（Claude Code Remote）的核心通信层。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `replBridge.ts` | 2406 | REPL Bridge 主逻辑：会话管理、消息收发、权限桥接 |
| `bridgeApi.ts` | ~400 | HTTP API 客户端：poll work、send events |
| `types.ts` | 262 | 协议类型定义 |
| `bridgeConfig.ts` | 48 | Bridge 配置与认证 |
| `initReplBridge.ts` | — | REPL 启动时的 Bridge 初始化 |
| `bridgeMessaging.ts` | — | 消息处理与转换 |

### 核心代码片段

**Bridge Handle** 是 REPL 与远程会话的接口：

```typescript
export type ReplBridgeHandle = {
  bridgeSessionId: string
  environmentId: string
  sessionIngressUrl: string
  writeMessages(messages: Message[]): void
  writeSdkMessages(messages: SDKMessage[]): void
  sendControlRequest(request: SDKControlRequest): void
  sendControlResponse(response: SDKControlResponse): void
  sendControlCancelRequest(requestId: string): void
  sendResult(): void
  teardown(): Promise<void>
}
```

**Work Secret** 是服务器下发的会话凭证：

```typescript
export type WorkSecret = {
  version: number
  session_ingress_token: string
  api_base_url: string
  sources: Array<{ type: string; git_info?: { ... } }>
  auth: Array<{ type: string; token: string }>
  use_code_sessions?: boolean
}
```

**Bridge API** 使用长轮询实现双向通信：

```typescript
export function createBridgeApiClient(deps: BridgeApiDeps): BridgeApiClient {
  // 401 处理：尝试 OAuth 刷新后重试一次
  // 安全验证：ID 注入防护
  function validateBridgeId(id: string, label: string): string {
    if (!id || !SAFE_ID_PATTERN.test(id)) {
      throw new Error(`Invalid ${label}: contains unsafe characters`)
    }
    return id
  }
}
```

### 设计要点
- **混合传输**：V1（长轮询）与 V2（WebSocket + 轮询 fallback）两种传输层
- **模块拆分**：`initReplBridge.ts` 与 `replBridge.ts` 分离，避免 `sessionStorage` 的重依赖图污染 Agent SDK bundle
- **FlushGate**：确保消息在批次边界才发送，避免部分状态泄露
- **CapacityWake**：当服务器指示有新工作时唤醒客户端
- **安全性**：所有服务器提供的 ID 经过 `validateBridgeId` 校验，防止路径遍历；Trusted Device Token 用于 ELEVATED 安全等级

---

## 4. assistant/ — 会话历史

### 功能概述
提供与 Anthropic API 的会话事件端点交互的能力，支持分页查询历史消息。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `sessionHistory.ts` | 87 | 会话历史分页查询 |

### 核心代码片段

```typescript
export const HISTORY_PAGE_SIZE = 100

export type HistoryPage = {
  events: SDKMessage[]
  firstId: string | null
  hasMore: boolean
}

export async function createHistoryAuthCtx(sessionId: string): Promise<HistoryAuthCtx> {
  const { accessToken, orgUUID } = await prepareApiRequest()
  return {
    baseUrl: `${getOauthConfig().BASE_API_URL}/v1/sessions/${sessionId}/events`,
    headers: {
      ...getOAuthHeaders(accessToken),
      'anthropic-beta': 'ccr-byoc-2025-07-29',
      'x-organization-uuid': orgUUID,
    },
  }
}

export async function fetchLatestEvents(ctx: HistoryAuthCtx, limit = HISTORY_PAGE_SIZE): Promise<HistoryPage | null>
```

### 设计要点
- **认证隔离**：`createHistoryAuthCtx` 一次性准备认证信息，后续页面复用
- **游标分页**：`firstId` 作为 `before_id` 游标，支持向前翻页
- **容错**：HTTP 错误返回 null 而非抛异常，调用者可优雅降级

---

## 5. state/ — 全局状态 Store

### 功能概述
实现了轻量级响应式状态管理，是整个 UI 层的数据基础。采用类似 Redux 的不可变更新 + 订阅模式，但更轻量。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `store.ts` | 34 | 通用 Store 工厂函数 |
| `AppStateStore.ts` | ~800 | AppState 类型定义与默认值 |
| `AppState.tsx` | 199 | React Provider，桥接 Store 与 React |
| `selectors.ts` | 76 | 派生状态选择器 |
| `bootstrap/state.ts` | 1758 | 全局命令式状态（非 React） |

### 核心代码片段

**Store 工厂**（store.ts）：

```typescript
export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等则跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**AppState Provider**（AppState.tsx）将 Store 与 React 集成：

```typescript
export function AppStateProvider({ children, initialState, onChangeAppState }: Props) {
  const [store] = useState(() => createStore(initialState ?? getDefaultAppState(), onChangeAppState))
  // 单例模式：HasAppStateContext 防止嵌套
  // 条件集成：VoiceProvider 仅在 VOICE_MODE feature 开启时包装
  return (
    <AppStoreContext.Provider value={store}>
      <MailboxProvider>
        <VoiceProvider>{children}</VoiceProvider>
      </MailboxProvider>
    </AppStoreContext.Provider>
  )
}
```

**Bootstrap State** 是命令式全局状态，在 React 之外使用：

```typescript
type State = {
  originalCwd: string
  projectRoot: string
  totalCostUSD: number
  totalAPIDuration: number
  isInteractive: boolean
  kairosActive: boolean
  strictToolResultPairing: boolean
  clientType: string
  // ... 100+ 字段
}
```

**Selectors** 提供纯函数派生：

```typescript
export function getActiveAgentForInput(appState: AppState): ActiveAgentForInput {
  const viewedTask = getViewedTeammateTask(appState)
  if (viewedTask) return { type: 'viewed', task: viewedTask }
  // ...
  return { type: 'leader' }
}
```

### 设计要点
- **双轨状态**：`bootstrap/state.ts`（命令式，模块级变量）用于启动和工具层；`AppStateStore`（React Store）用于 UI 层
- **引用相等优化**：`Object.is(next, prev)` 避免无意义更新
- **React Compiler**：编译后的代码使用 `_c(n)` 缓存槽，手动优化渲染
- **防嵌套**：`HasAppStateContext` 防止多层 Provider 嵌套

---

## 6. context/ — React Context

### 功能概述
提供 React Context 用于跨组件传递状态和服务，包括邮箱消息队列、排队消息布局、通知系统和语音状态。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `mailbox.tsx` | 37 | 邮箱消息队列 Provider |
| `QueuedMessageContext.tsx` | 62 | 排队消息布局上下文 |
| `notifications.tsx` | 239 | 通知系统（优先级、折叠、队列） |
| `voice.tsx` | 87 | 语音状态 Provider |

### 核心代码片段

**Mailbox** 是消息传递原语：

```typescript
export function MailboxProvider({ children }: Props) {
  const mailbox = useMemo(() => new Mailbox(), [])
  return <MailboxContext.Provider value={mailbox}>{children}</MailboxContext.Provider>
}
```

**通知系统** 支持优先级和折叠：

```typescript
type Priority = 'low' | 'medium' | 'high' | 'immediate'
type BaseNotification = {
  key: string
  invalidates?: string[]      // 使其他通知失效
  priority: Priority
  timeoutMs?: number
  fold?: (acc: Notification, incoming: Notification) => Notification  // 合并同类通知
}
```

**语音状态** 使用独立 Store：

```typescript
export type VoiceState = {
  voiceState: 'idle' | 'recording' | 'processing'
  voiceError: string | null
  voiceInterimTranscript: string
  voiceAudioLevels: number[]
  voiceWarmingUp: boolean
}
```

### 设计要点
- **通知折叠**（fold）：类似 Array.reduce 的合并策略，避免同类通知重复
- **失效机制**（invalidates）：新通知可声明哪些旧通知应立即清除
- **immediate 优先级**：立即清除当前通知并显示，用于紧急信息

---

## 7. cli/ — CLI 定义

### 功能概述
CLI 层是整个系统的输入/输出枢纽。`print.ts` 是主入口（近 5600 行），负责 REPL 初始化、工具池组装、消息循环和命令分发。`structuredIO.ts` 为 SDK 模式提供结构化 JSON I/O。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `print.ts` | 5594 | CLI 主入口、REPL 循环 |
| `structuredIO.ts` | 859 | SDK 模式结构化 I/O |
| `exit.ts` | 31 | 统一退出辅助函数 |

### 核心代码片段

**CLI 退出辅助**（exit.ts）统一了 60+ 处 `console.error + process.exit`：

```typescript
export function cliError(msg?: string): never {
  if (msg) console.error(msg)
  process.exit(1)
  return undefined as never  // 让 TS 在调用处做控制流窄化
}

export function cliOk(msg?: string): never {
  if (msg) process.stdout.write(msg + '\n')
  process.exit(0)
  return undefined as never
}
```

**StructuredIO** 处理 SDK 模式的 JSON-RPC 通信：

```typescript
export class StructuredIO {
  // 通过 stdin/stdout 的 NDJSON 协议与 SDK 宿主通信
  // 处理权限请求、钩子输入/输出、控制消息
}
```

### 设计要点
- **print.ts 巨型模块**：作为入口承担了过多职责（初始化 + 工具组装 + 消息循环），但拆分风险高
- **统一退出**：`cliError/cliOk` 的 `never` 返回类型让调用处可安全省略后续代码
- **SDK vs REPL 双模式**：`StructuredIO` 和 `RemoteIO` 分别处理 SDK 和远程模式的 I/O

---

## 8. tasks/ — 任务管理器

### 功能概述
任务系统管理所有后台/前台异步操作，包括 shell 任务、agent 任务、远程 agent 任务、teammate 任务和 dream（记忆整理）任务。每个任务类型有独立的状态机。

### 关键文件
| 文件 | 职责 |
|------|------|
| `types.ts` | 任务类型联合与后台任务判定 |
| `DreamTask/` | 记忆整理后台任务 |
| `InProcessTeammateTask/` | 进程内 teammate 任务 |
| `LocalAgentTask/` | 本地 agent 任务 |
| `LocalShellTask/` | 本地 shell 任务 |
| `RemoteAgentTask/` | 远程 agent 任务 |
| `LocalWorkflowTask/` | 工作流任务 |
| `MonitorMcpTask/` | MCP 监控任务 |
| `stopTask.ts` | 任务停止逻辑 |
| `pillLabel.ts` | 任务状态显示标签 |

### 核心代码片段

**任务类型联合**：

```typescript
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState

export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') return false
  if ('isBackgrounded' in task && task.isBackgrounded === false) return false
  return true
}
```

**InProcessTeammateTask** 是最复杂的任务类型：

```typescript
export type InProcessTeammateTaskState = TaskStateBase & {
  type: 'in_process_teammate'
  identity: TeammateIdentity       // agentId, agentName, teamName, color
  prompt: string
  model?: string
  selectedAgent?: AgentDefinition
  abortController?: AbortController
  currentWorkAbortController?: AbortController  // 中止单轮而非整个 teammate
}
```

**DreamTask** 记忆整理后台任务：

```typescript
export type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: DreamPhase               // 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]          // 不完整——只捕获直接工具调用
  turns: DreamTurn[]              // 最近 30 轮
  abortController?: AbortController
  priorMtime: number              // 用于 kill 时回滚 consolidation lock
}
```

### 设计要点
- **判别联合**：`TaskState` 通过 `type` 字段区分，TypeScript 可穷举处理
- **前台/后台区分**：`isBackgrounded` 标志控制 UI 是否在 footer pill 显示
- **Abort 双层**：teammate 有两个 AbortController——整体终止和单轮中止
- **注册表模式**：`registerTask/updateTaskState` 全局管理任务生命周期

---

## 9. remote/ — 远程会话

### 功能概述
管理 CCR（Claude Code Remote）的远程会话连接。通过 WebSocket 与服务器保持实时双向通信，处理消息转发、权限审批和会话恢复。

### 关键文件
| 文件 | 职责 |
|------|------|
| `RemoteSessionManager.ts` | 远程会话生命周期管理 |
| `SessionsWebSocket.ts` | WebSocket 连接与自动重连 |
| `remotePermissionBridge.ts` | 远程权限请求桥接 |
| `sdkMessageAdapter.ts` | SDK 消息格式转换 |

### 核心代码片段

**WebSocket 重连策略**：

```typescript
const RECONNECT_DELAY_MS = 2000
const MAX_RECONNECT_ATTEMPTS = 5
const MAX_SESSION_NOT_FOUND_RETRIES = 3  // compaction 期间可能是暂时的

const PERMANENT_CLOSE_CODES = new Set([4003])  // unauthorized

type WebSocketState = 'connecting' | 'connected' | 'closed'
```

**消息类型过滤**：

```typescript
function isSDKMessage(message): message is SDKMessage {
  return message.type !== 'control_request'
    && message.type !== 'control_response'
    && message.type !== 'control_cancel_request'
}
```

**权限响应**：

```typescript
export type RemotePermissionResponse =
  | { behavior: 'allow'; updatedInput: Record<string, unknown> }
  | { behavior: 'deny'; message: string }
```

### 设计要点
- **分层重连**：4001（session not found）允许 3 次重试（compaction 期间暂态），4003 直接放弃
- **WebSocket + mTLS**：使用 TLS 客户端证书认证
- **消息适配**：`sdkMessageAdapter` 在 SDK 格式和远程格式间转换

---

## 10. voice/ — 语音交互

### 功能概述
语音模式的开关控制。通过 GrowthBook feature flag 和 OAuth 认证双重门控。

### 关键文件
| 文件 | 职责 |
|------|------|
| `voiceModeEnabled.ts` | 语音模式启用判定 |

### 核心代码片段

```typescript
export function isVoiceModeEnabled(): boolean {
  return hasVoiceAuth() && isVoiceGrowthBookEnabled()
}

export function hasVoiceAuth(): boolean {
  if (!isAnthropicAuthEnabled()) return false
  const tokens = getClaudeAIOAuthTokens()
  return Boolean(tokens?.accessToken)
}

export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_quartz_disabled', false)
    : false
}
```

### 设计要点
- **双重门控**：feature flag（编译时裁剪） + GrowthBook kill-switch（运行时远程） + OAuth 认证
- **Positive ternary 模式**：`feature('X') ? ... : false` 确保外部构建裁剪掉 inline 字符串
- **仅限 Anthropic OAuth**：voice_stream 端点不支持 API key、Bedrock、Vertex

---

## 11. vim/ — Vim 模式

### 功能概述
为编辑器组件实现了完整的 Vim 模式状态机。支持 Normal/Insert 模式、operator（d/c/y）、motion、text object 和 dot-repeat。

### 关键文件
| 文件 | 职责 |
|------|------|
| `types.ts` | 状态机类型定义（完整状态图） |
| `motions.ts` | 光标移动纯函数 |
| `operators.ts` | 操作执行（删除、修改、yank 等） |
| `transitions.ts` | 状态转移表 |
| `textObjects.ts` | text object 解析 |

### 核心代码片段

**状态机类型**（types.ts 中的 ASCII 状态图）：

```
*  NORMAL
*         idle ──┬─[d/c/y]──► operator
*                 ├─[1-9]────► count
*                 ├─[fFtT]───► find
*                 ├─[g]──────► g
*                 ├─[r]──────► replace
*                 └─[><]─────► indent
*
*         operator ─┬─[motion]──► execute
*                    ├─[0-9]────► operatorCount
*                    ├─[ia]─────► operatorTextObj
*                    └─[fFtT]───► operatorFind
```

**Motion 纯函数**（motions.ts）：

```typescript
export function resolveMotion(key: string, cursor: Cursor, count: number): Cursor {
  let result = cursor
  for (let i = 0; i < count; i++) {
    const next = applySingleMotion(key, result)
    if (next.equals(result)) break  // 到达边界停止
    result = next
  }
  return result
}
```

**转移函数**（transitions.ts）：

```typescript
export function transition(state: CommandState, input: string, ctx: TransitionContext): TransitionResult {
  // 基于 state.type 分派到对应处理函数
  // 返回 { next?: CommandState, execute?: () => void }
}
```

### 设计要点
- **类型即文档**：`CommandState` 联合类型精确描述每个状态期望的输入
- **纯函数**：motion 和 operator 不修改状态，只返回计算结果
- **状态转移表**：集中定义所有转移，可扫描性极强
- **count 叠加**：支持数字前缀（如 `3dw`），通过循环应用实现

---

## 12. buddy/ — AI 伴侣

### 功能概述
AI 伴侣系统为用户在输入框旁生成一个随机小动物角色。角色有稀有度、属性、外观，会偶尔在气泡中评论。

### 关键文件
| 文件 | 职责 |
|------|------|
| `companion.ts` | 角色生成（确定性随机、稀有度、属性） |
| `types.ts` | 物种、帽子、眼睛、稀有度、属性类型 |
| `prompt.ts` | 伴侣提示词注入 |
| `sprites.ts` | 精灵图定义 |
| `CompanionSprite.tsx` | 角色渲染组件 |
| `useBuddyNotification.tsx` | 伴侣通知钩子 |

### 核心代码片段

**确定性角色生成**：

```typescript
function mulberry32(seed: number): () => number { ... }
function hashString(s: string): number {
  if (typeof Bun !== 'undefined') return Number(BigInt(Bun.hash(s)) & 0xffffffffn)
  // FNV-1a fallback
}

function rollRarity(rng: () => number): Rarity {
  const total = Object.values(RARITY_WEIGHTS).reduce((a, b) => a + b, 0)
  let roll = rng() * total
  for (const rarity of RARITIES) {
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'
}
```

**属性分配**（稀有度影响下限）：

```typescript
const RARITY_FLOOR: Record<Rarity, number> = {
  common: 5, uncommon: 15, rare: 25, epic: 35, legendary: 50,
}
// 一个 peak stat、一个 dump stat，其余随机
```

**伴侣提示词**：

```typescript
export function companionIntroText(name: string, species: string): string {
  return `A small ${species} named ${name} sits beside the user's input box...
When the user addresses ${name} directly (by name), its bubble will answer.
Your job in that moment is to stay out of the way.`
}
```

### 设计要点
- **确定性随机**：基于用户 ID 哈希的 seeded PRNG，同一用户每次看到相同角色
- **Gacha 机制**：稀有度权重、属性分配模拟了抽卡游戏的乐趣
- **模型隔离**：伴侣由独立系统处理，主模型收到指令"不要替伴侣说话"

---

## 13. bootstrap/ — 引导系统

### 功能概述
`bootstrap/state.ts`（1758 行）是整个应用的命令式全局状态中心。它在 React 之外运作，存储会话 ID、项目路径、成本统计、模型配置等跨系统共享的状态。

### 核心代码片段

```typescript
type State = {
  originalCwd: string
  projectRoot: string
  totalCostUSD: number
  totalAPIDuration: number
  isInteractive: boolean
  kairosActive: boolean
  strictToolResultPairing: boolean
  clientType: string
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  // ... 100+ 字段
}
```

### 设计要点
- **巨型模块**：1758 行，文件头部注释 `DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE`
- **双轨架构**：命令式 `bootstrap/state.ts` 用于非 React 代码，React Store 用于 UI
- **间接导入**：通过注释标注的 `eslint-disable` 绕过 bootstrap 隔离规则

---

## 14. memdir/ — Memory 目录管理

### 功能概述
管理 `MEMORY.md` 和记忆文件的读写、截断和类型分类。记忆系统帮助 Claude 跨会话保留关键信息。

### 关键文件
| 文件 | 职责 |
|------|------|
| `memdir.ts` | MEMORY.md 读写与截断 |
| `paths.ts` | 记忆文件路径解析与开关判定 |
| `memoryTypes.ts` | 记忆类型分类（user/feedback/project/reference） |
| `findRelevantMemories.ts` | 记忆搜索 |
| `memoryScan.ts` | 记忆目录扫描 |
| `teamMemPaths.ts` | 团队记忆路径 |
| `teamMemPrompts.ts` | 团队记忆提示词 |

### 核心代码片段

**记忆开关链**（paths.ts）：

```typescript
export function isAutoMemoryEnabled(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY)) return false
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) && !process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) return false
  const settings = getInitialSettings()
  if (settings.autoMemoryEnabled !== undefined) return settings.autoMemoryEnabled
  return true  // 默认启用
}
```

**MEMORY.md 截断**：

```typescript
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // 先按行截断（自然边界），再按字节截断（在最后一个换行符处）
}
```

**记忆类型分类**（memoryTypes.ts）：

```
- user: 用户角色、偏好、知识
- feedback: 用户对 AI 行为的纠正和确认
- project: 项目上下文（不可从代码/git 派生的信息）
- reference: 代码片段、API 示例等参考资料

每种类型有 `<scope>` 标签区分 `private`（仅个人）和 `team`（团队共享）。

### 设计要点
- **多级开关**：环境变量 → simple 模式 → remote 模式 → settings.json → 默认启用
- **双重截断**：行数 + 字节数，防止超长行绕过行数限制
- **记忆类型严格分类**：明确禁止保存可从代码派生的信息（如架构、git 历史）
- **私人/团队分层**：user 类型始终 private，project 偏向 team

---

## 15. server/ — 服务端

### 功能概述
提供 Direct Connect 模式，允许外部客户端直接连接到 Claude Code 实例创建会话。

### 关键文件
| 文件 | 职责 |
|------|------|
| `createDirectConnectSession.ts` | 创建 Direct Connect 会话 |
| `directConnectManager.ts` | 连接管理器 |
| `types.ts` | 连接响应 schema |

### 核心代码片段

```typescript
export class DirectConnectError extends Error { ... }

export async function createDirectConnectSession({
  serverUrl, authToken, cwd, dangerouslySkipPermissions,
}): Promise<{ config: DirectConnectConfig; workDir?: string }> {
  const headers: Record<string, string> = { 'content-type': 'application/json' }
  if (authToken) headers['authorization'] = `Bearer ${authToken}`

  const resp = await fetch(`${serverUrl}/sessions`, {
    method: 'POST', headers,
    body: jsonStringify({ cwd, ...(dangerouslySkipPermissions && { dangerously_skip_permissions: true }) }),
  })

  const result = connectResponseSchema().safeParse(await resp.json())
  if (!result.success) throw new DirectConnectError(`Invalid session response: ${result.error.message}`)
}
```

### 设计要点
- **Zod 验证**：服务器响应经过 schema 校验，防止格式不符
- **可选权限跳过**：`dangerouslySkipPermissions` 通过请求体传递
- **错误分类**：`DirectConnectError` 区分网络、HTTP、解析三类失败

---

## 16. types/ — 类型定义

### 功能概述
共享类型定义模块，包含消息类型、ID 类型、钩子类型等跨模块使用的类型。

### 关键类型
- **消息类型**：`Message`, `AssistantMessage`, `NormalizedUserMessage` 等
- **ID 类型**：`SessionId`, `TaskId` 等 branded types
- **钩子类型**：`HookCallbackMatcher`, `HookEvent` 等
- **命令类型**：`Command`, `PromptCommand` 等
- **插件类型**：`LoadedPlugin`, `BuiltinPluginDefinition` 等

### 设计要点
- 使用 TypeScript branded types 防止 ID 类型混用
- 类型文件按领域划分，避免循环依赖

---

## 17. constants/ — 常量

### 功能概述
集中管理全局常量，包括 OAuth 配置、输出样式常量、模型名称、权限常量等。约 21 个文件覆盖各子系统的配置常量。

### 设计要点
- 按功能域拆分（oauth、outputStyles、permissions 等）
- 常量文件通常是模块图的叶子节点，无副作用

---

## 18. migrations/ — 配置迁移

### 功能概述
处理用户配置的版本迁移，确保从旧版本升级时配置格式兼容。

### 关键迁移
| 文件 | 迁移内容 |
|------|----------|
| `migrateAutoUpdatesToSettings.ts` | 自动更新偏好 → settings.json env |
| `migrateBypassPermissionsAcceptedToSettings.ts` | 绕过权限确认 → settings.json |
| `migrateFennecToOpus.ts` | Fennec 模型 → Opus |
| `migrateSonnet1mToSonnet45.ts` | Sonnet 1m → Sonnet 4.5 |
| `migrateSonnet45ToSonnet46.ts` | Sonnet 4.5 → Sonnet 4.6 |
| `migrateReplBridgeEnabledToRemoteControlAtStartup.ts` | REPL Bridge → Remote Control |
| `resetProToOpusDefault.ts` | Pro 默认模型重置 |

### 核心模式

```typescript
export function migrateAutoUpdatesToSettings(): void {
  const globalConfig = getGlobalConfig()
  if (globalConfig.autoUpdates !== false || ...) return

  updateSettingsForSource('userSettings', {
    ...userSettings,
    env: { ...userSettings.env, DISABLE_AUTOUPDATER: '1' },
  })

  // 迁移完成后从旧位置清除
  saveGlobalConfig(current => {
    const { autoUpdates: _, ...updatedConfig } = current
    return updatedConfig
  })

  logEvent('tengu_migrate_autoupdates_to_settings', { ... })
}
```

### 设计要点
- **幂等性**：每个迁移检查前置条件，已迁移则跳过
- **双向记录**：迁移完成后从旧位置删除 + 记录 analytics
- **模型重命名**：多个迁移处理模型名称变更，保持用户选择有效

---

## 19. keybindings/ — 键绑定

### 功能概述
完整可自定义的键绑定系统。支持单键、组合键（chord）、上下文绑定、用户自定义覆盖。

### 关键文件
| 文件 | 职责 |
|------|------|
| `defaultBindings.ts` | 默认键绑定定义 |
| `parser.ts` | 按键字符串解析（"ctrl+shift+k" → 结构化） |
| `resolver.ts` | 按键匹配与动作解析 |
| `match.ts` | 按键匹配逻辑 |
| `schema.ts` | 键绑定配置 schema |
| `loadUserBindings.ts` | 用户自定义绑定加载 |
| `reservedShortcuts.ts` | 不可重定义的快捷键 |
| `validate.ts` | 绑定冲突检测 |
| `shortcutFormat.ts` | 快捷键显示格式化 |
| `template.ts` | 绑定模板 |

### 核心代码片段

**按键解析**（parser.ts）：

```typescript
export function parseKeystroke(input: string): ParsedKeystroke {
  const parts = input.split('+')
  // 支持 ctrl/control, alt/opt/option/meta, cmd/command/super/win
  // 特殊键: esc→escape, return→enter
}
```

**上下文绑定解析**（resolver.ts）：

```typescript
export function resolveKey(
  input: string, key: Key,
  activeContexts: KeybindingContextName[],
  bindings: ParsedBinding[],
): ResolveResult {
  // 最后一个匹配的绑定胜出（用户覆盖默认）
  for (const binding of bindings) {
    if (binding.chord.length !== 1) continue
    if (!ctxSet.has(binding.context)) continue
    if (matchesBinding(input, key, binding)) match = binding
  }
}
```

**平台适配**（defaultBindings.ts）：

```typescript
const IMAGE_PASTE_KEY = getPlatform() === 'windows' ? 'alt+v' : 'ctrl+v'
const MODE_CYCLE_KEY = SUPPORTS_TERMINAL_VT_MODE ? 'shift+tab' : 'meta+m'
```

### 设计要点
- **上下文感知**：同一按键在不同上下文（Global、Chat、Editor）绑定不同动作
- **用户覆盖**：用户绑定后加载，last-match-wins 策略覆盖默认
- **平台差异**：Windows 上 ctrl+v 是系统粘贴，改用 alt+v
- **Chord 支持**：支持多键组合（如 `g g` 跳到文件头）
- **保留键**：ctrl+c/ctrl+d 使用基于时间的双击检测，不可重绑定

---

## 20. entrypoints/ — 入口点

### 功能概述
定义了多个入口点，分别用于 CLI 启动、Agent SDK 和类型导出。

### 关键文件
| 文件 | 职责 |
|------|------|
| `agentSdkTypes.ts` | Agent SDK 公共类型导出 |
| `cli.ts` | CLI 入口 |
| `sdk/` | SDK 子模块（coreTypes, runtimeTypes, controlTypes, toolTypes） |

### 核心代码片段

**Agent SDK 类型入口**：

```typescript
// Agent SDK 公共 API：从子模块 re-export
export * from './sdk/coreTypes.js'       // 可序列化类型
export * from './sdk/runtimeTypes.js'    // 回调、接口
export type { Settings } from './sdk/settingsTypes.generated.js'

// SDK tool 函数
export function tool<Schema extends AnyZodRawShape>(
  _name: string, _description: string, _inputSchema: Schema,
  _handler: (args: InferShape<Schema>, extra: unknown) => Promise<CallToolResult>,
)
```

### 设计要点
- **多入口**：CLI 入口、SDK 入口、类型入口分离，支持 tree-shaking
- **分层导出**：coreTypes（可序列化）与 runtimeTypes（有方法）分开

---

## 21. screens/ — 屏幕

### 功能概述
定义了三个全屏界面组件。

### 关键文件
| 文件 | 职责 |
|------|------|
| `REPL.tsx` | 主 REPL 界面 |
| `WelcomeScreen.tsx` | 欢迎屏幕 |
| `ResumeConversation.tsx` | 恢复对话界面 |
| `Doctor.tsx` | 诊断界面 |

---

## 22. upstreamproxy/ — 上游代理

### 功能概述
CCR 容器内的 MITM 代理，为 agent 子进程的出站请求注入组织级凭证（如 Datadog API Key）。通过 WebSocket tunnel 连接到 CCR 服务端的 upstreamproxy endpoint。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `relay.ts` | 455 | localhost TCP → WebSocket tunnel |
| `upstreamproxy.ts` | 285 | 容器侧初始化 |

### 核心代码片段

**初始化流程**（upstreamproxy.ts）：

```typescript
// 1. 读取 session token: /run/ccr/session_token
// 2. prctl(PR_SET_DUMPABLE, 0) 防止 ptrace
// 3. 下载 MITM CA 证书 → 合并到系统 bundle
// 4. 启动 CONNECT→WebSocket relay
// 5. 删除 token 文件（token 仅在堆内存中）
// 6. 设置 HTTPS_PROXY / SSL_CERT_FILE 环境变量

const NO_PROXY_LIST = [
  'localhost', '127.0.0.1', '::1',
  '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',  // RFC1918
  '169.254.169.254/32',  // IMDS
  // npm/PyPI/GitHub 等已有直连的域名
]
```

**CONNECT-over-WebSocket 协议**：

```typescript
// TCP accept HTTP CONNECT → bytes 包装为 UpstreamProxyChunk protobuf → WebSocket 发送
// 服务端解包 → MITM TLS → 注入凭证 → 转发到真实上游
```

### 设计要点
- **Fail-open**：任何步骤失败只记录警告，不阻断正常会话
- **Token 单次使用**：文件在 relay 启动后立即删除，token 仅堆内存持有
- **NO_PROXY**：环回地址、内网、IMDS、包管理器域名不经过代理
- **安全加固**：`PR_SET_DUMPABLE=0` 防止同 UID 进程 ptrace 读取堆内存

---

## 23. native-ts/ — Native 模块 TypeScript 实现

### 功能概述
为三个 Rust NAPI 原生模块提供了纯 TypeScript fallback 实现，在不支持原生模块的环境中提供相同 API。

### 关键文件
| 目录 | 原生对应 | 职责 |
|------|----------|------|
| `color-diff/` | vendor/color-diff-src (syntect+bat) | 语法高亮 + word diff |
| `file-index/` | vendor/file-index-src (nucleo) | 模糊文件搜索 |
| `yoga-layout/` | yoga-layout (Meta Flexbox) | Flexbox 布局引擎 |

### 核心代码片段

**File Index**（file-index/index.ts）模拟 nucleo 评分：

```typescript
const SCORE_MATCH = 16
const BONUS_BOUNDARY = 8
const BONUS_CAMEL = 6
const BONUS_CONSECUTIVE = 4
const BONUS_FIRST_CHAR = 8
const PENALTY_GAP_START = 3
const PENALTY_GAP_EXTENSION = 1

// 包含 "test" 的路径 1.05× 惩罚（上限 1.0）
```

**Yoga Layout**（yoga-layout/index.ts）实现了简化版 Flexbox：

```typescript
// 上游 C++ CalculateLayout.cpp ~2500 行
// TS 移植仅覆盖 Ink 实际使用的子集
```

**Color Diff** 惰性加载 highlight.js：

```typescript
// 延迟到首次渲染才加载 highlight.js（~50MB, 190+ 语言语法）
// 避免 module-eval 时阻塞
```

### 设计要点
- **API 兼容**：TS 实现完全匹配原生模块的接口签名
- **性能权衡**：TS 版本慢于原生，但零编译依赖
- **惰性加载**：highlight.js 延迟加载，避免启动时间惩罚

---

## 24. query/ — 查询引擎辅助

### 功能概述
为 Claude Code 的主查询循环提供配置、依赖注入、停止钩子和 token 预算管理。

### 关键文件
| 文件 | 行数 | 职责 |
|------|------|------|
| `config.ts` | 46 | 查询配置快照（不可变） |
| `deps.ts` | 40 | 依赖注入（测试可 mock） |
| `stopHooks.ts` | 473 | 停止钩子处理 |
| `tokenBudget.ts` | 93 | Token 预算跟踪与继续决策 |

### 核心代码片段

**依赖注入**（deps.ts）：

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming
  microcompact: typeof microcompactMessages
  autocompact: typeof autoCompactIfNeeded
  uuid: () => string
}

export function productionDeps(): QueryDeps { ... }
```

**Token 预算**（tokenBudget.ts）：

```typescript
export type BudgetTracker = {
  continuationCount: number
  lastDeltaTokens: number
  lastGlobalTurnTokens: number
  startedAt: number
}

const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

// 继续决策基于完成百分比 + 收益递减检测
```

### 设计要点
- **不可变配置**：`QueryConfig` 在 `query()` 入口快照一次，后续不变
- **可测试性**：`QueryDeps` 让测试直接注入 fake，无需 spyOn
- **收益递减**：当 token 产出低于阈值时自动停止，避免浪费

---

## 25. schemas/ — Schema 定义

### 功能概述
从 `utils/settings/types.ts` 提取的 Hook Zod schema，用于打破循环依赖。

### 关键文件
| 文件 | 职责 |
|------|------|
| `hooks.ts` | Hook 相关 Zod schema（222 行） |

### 核心代码片段

```typescript
// 支持 Bash command hook 和 URL hook 两种类型
// IfConditionSchema 使用权限规则语法过滤钩子触发
const IfConditionSchema = lazySchema(() =>
  z.string().optional().describe('Permission rule syntax (e.g., "Bash(git *)")')
)
```

### 设计要点
- **循环依赖打破**：`settings/types.ts` 和 `plugins/schemas.ts` 都从此文件导入
- **Lazy schema**：使用 `lazySchema` 延迟 schema 构建避免启动时开销

---

## 26. outputStyles/ — 输出样式

### 功能概述
从 `.claude/output-styles/` 目录加载 Markdown 文件作为输出样式配置，允许用户自定义输出格式。

### 关键文件
| 文件 | 职责 |
|------|------|
| `loadOutputStylesDir.ts` | 从目录加载输出样式（98 行） |

### 核心机制

```typescript
// 项目 .claude/output-styles/*.md → 项目样式
// 用户 ~/.claude/output-styles/*.md → 用户样式（被项目样式覆盖）
// 文件名 → 样式名，内容 → 样式提示词，frontmatter → 名称和描述
```

### 设计要点
- **项目/用户分层**：项目级样式覆盖用户级
- **Markdown 配置**：与 skills/commands 使用相同的加载基础设施
- **Memoize**：结果缓存，避免重复文件系统扫描

---

## 27. moreright/ — 扩展权限

### 功能概述
扩展权限模块，用于处理需要更细粒度权限控制的场景。（源码目录为空或文件极少）

### 设计要点
- 名称暗示 "more right"（更多权限），可能是实验性功能模块

---

## 总结

### 架构全景

这些模块共同构成了 Claude Code 的基础设施层：

| 层次 | 模块 | 角色 |
|------|------|------|
| **状态管理** | state/, bootstrap/, context/ | 全局状态、React Context |
| **通信** | bridge/, remote/, server/ | 远程控制、WebSocket、Direct Connect |
| **扩展** | skills/, plugins/, schemas/ | 技能、插件、配置 schema |
| **交互** | cli/, keybindings/, vim/, voice/ | CLI、键绑定、编辑器模式、语音 |
| **任务** | tasks/ | 后台任务管理 |
| **记忆** | memdir/ | 跨会话记忆持久化 |
| **基础设施** | entrypoints/, types/, constants/, migrations/ | 入口、类型、常量、迁移 |
| **UI** | screens/, buddy/ | 界面组件、AI 伴侣 |
| **平台** | native-ts/, upstreamproxy/ | 原生模块 fallback、代理 |
| **查询** | query/, outputStyles/ | 查询辅助、输出样式 |

### 关键设计模式

1. **判别联合**（tasks, vim, notifications）：TypeScript 类型安全的状态机
2. **注册表模式**（plugins, skills, mcpSkillBuilders）：解耦注册与使用
3. **双轨状态**（bootstrap/state + AppStateStore）：命令式 vs 响应式
4. **Fail-open**（upstreamproxy, bridge）：错误不阻断核心功能
5. **循环依赖打破**（schemas/, mcpSkillBuilders.ts）：提取共享依赖
6. **确定性随机**（buddy）：seeded PRNG 保证可重现
7. **惰性加载**（native-ts/color-diff）：延迟重型依赖的加载时机