# Claude Code Harness Engineering 设计深度分析

## 1. 什么是 "Harness"

在 Claude Code 的源码中，"Harness" 并非单一模块，而是**宿主执行环境**的统称——负责运行、控制、监控 Claude Code Agent 进程的基础设施层。

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Harness 工程全景                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Layer 1: Entry Point Harness（入口层）                     │     │
│  │  cli.tsx → main.tsx → REPL / SDK / Headless               │     │
│  │  • Ablation baseline (L0 实验)                              │     │
│  │  • 环境变量控制 (DISABLE_*)                                  │     │
│  │  • Feature flag DCE (死代码消除)                             │     │
│  └────────────────────────────────────────────────────────────┘     │
│                              │                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Layer 2: Bridge/CCR Harness（远程控制层）                   │     │
│  │  bridge/ → bridgeMain → sessionRunner → workSecret          │     │
│  │  • Remote Control (claude.ai → 本地 Agent)                 │     │
│  │  • CCR v1 Session Ingress / v2 Internal Event Writer       │     │
│  │  • 多会话调度（--spawn / --capacity）                       │     │
│  └────────────────────────────────────────────────────────────┘     │
│                              │                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Layer 3: Forked Agent Harness（子 Agent 基础设施）         │     │
│  │  forkedAgent.ts → CacheSafeParams → runForkedAgent          │     │
│  │  • Prompt Cache 共享                                        │     │
│  │  • 用量追踪                                                  │     │
│  │  • 状态隔离                                                  │     │
│  └────────────────────────────────────────────────────────────┘     │
│                              │                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Layer 4: Speculation Harness（推测执行层）                  │     │
│  │  speculation.ts → overlay filesystem → speculation accept   │     │
│  │  • 用户输入预测执行                                          │     │
│  │  • Overlay FS 写入隔离                                      │     │
│  │  • Accept/Deny 提交机制                                     │     │
│  └────────────────────────────────────────────────────────────┘     │
│                              │                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Layer 5: Hint Protocol（信号协议层）                        │     │
│  │  claudeCodeHints.ts → extractClaudeCodeHints → install prompt│   │
│  │  • <claude-code-hint /> 标签协议                            │     │
│  │  • Harness-only side channel                                │     │
│  │  • Plugin 推荐机制                                          │     │
│  └────────────────────────────────────────────────────────────┘     │
│                              │                                      │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │  Layer 6: Observability Harness（可观测性层）                │     │
│  │  growthbook.ts → analytics → datadog → sessionIngress       │     │
│  │  • Feature flag eval (eval harness 支持)                    │     │
│  │  • 环境变量覆盖 (eval harness 确定性)                       │     │
│  │  • A/B 实验基线                                             │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Layer 1: 入口点 Harness

### 2.1 Feature Flag 死代码消除（DCE）

> 文件：`entrypoints/cli.tsx`

Claude Code 使用 Bun 编译时的 feature flag 进行死代码消除。在编译时，不可用的 feature 代码被完全移除：

```typescript
// Dead code elimination: conditional imports for feature-gated modules
const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js')
    : null

const BRIEF_PROACTIVE_SECTION: string | null =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (require('../tools/BriefTool/prompt.js')).BRIEF_PROACTIVE_SECTION
    : null
```

**设计价值**：
- 外部构建完全移除内部功能代码（安全 + 体积）
- 运行时零开销（不是 `if` 判断，是编译时消除）
- 代码库统一，构建产物不同

### 2.2 Ablation Baseline

```typescript
// Harness-science L0 ablation baseline. Inlined here (not init.ts) because
// init.ts is shared with the SDK path and we want this to only trigger for CLI.
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  // L0 实验：禁用所有增强功能，测量基线性能
  process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY = '1'
  process.env.CLAUDE_CODE_DISABLE_THINKING = '1'
  // ... 更多 DISABLE_*
}
```

**Ablation 实验设计**：通过级联的环境变量禁用功能，对比有/无增强的 Agent 表现。

### 2.3 环境变量控制矩阵

| 环境变量 | 作用 | Harness 层 |
|----------|------|-----------|
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 禁用自动记忆 | Feature |
| `CLAUDE_CODE_DISABLE_THINKING` | 禁用思考模式 | Feature |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` | 禁用后台任务 | Feature |
| `CLAUDE_CODE_DISABLE_AUTO_COMPACT` | 禁用自动压缩 | Feature |
| `CLAUDE_CODE_DISABLE_SPECULATION` | 禁用推测执行 | Speculation |
| `CLAUDE_CODE_SIMPLE` | 简化模式（--bare） | Entry |
| `CLAUDE_CODE_REMOTE` | 远程模式标识 | Bridge |
| `CLAUDE_CODE_REMOTE_MEMORY_DIR` | 远程记忆持久化 | Bridge |

---

## 3. Layer 2: Bridge/CCR 远程控制

### 3.1 Bridge 架构

Bridge 是 Claude Code 的**远程控制基础设施**，允许 claude.ai 网页端控制本地 Agent：

```
┌──────────────────────────────────────────────────────────────────┐
│                    Bridge 远程控制架构                             │
│                                                                  │
│  claude.ai 网页                                                   │
│      │                                                           │
│      │  HTTPS (Work Secret)                                      │
│      ▼                                                           │
│  ┌──────────────────┐                                            │
│  │  CCR (Cloud Compute Runtime)                                  │
│  │  ┌────────────┐  ┌──────────────┐  ┌───────────────┐        │
│  │  │ v1 Ingress │  │ v2 EventWriter│  │ Capacity Wake │        │
│  │  └──────┬─────┘  └──────┬───────┘  └───────┬───────┘        │
│  └─────────┼───────────────┼──────────────────┼────────────────┘
│            │               │                  │                  │
│            └───────────────┼──────────────────┘                  │
│                            │ SSE / WebSocket                     │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  本地 Claude Code Agent (bridgeMain)                      │   │
│  │  ┌─────────────┐  ┌───────────────┐  ┌───────────────┐  │   │
│  │  │ Poll Loop   │  │ Session Runner│  │ Token Refresh │  │   │
│  │  │ (长轮询)    │  │ (spawn子进程) │  │ (JWT自动续期) │  │   │
│  │  └─────────────┘  └───────────────┘  └───────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 Work Secret 协议

```typescript
type WorkSecret = {
  version: number
  session_ingress_token: string  // 会话入口令牌
  api_base_url: string           // API 基础 URL
  sources: Array<{
    type: string
    git_info?: { type: string; repo: string; ref?: string; token?: string }
  }>
  auth: Array<{ type: string; token: string }>
  claude_code_args?: Record<string, string> | null  // CLI 参数
  mcp_config?: unknown | null                        // MCP 配置
  environment_variables?: Record<string, string> | null  // 环境变量
  use_code_sessions?: boolean                        // v2 选择器
}
```

**安全设计**：
- Work Secret 是 base64url 编码的 JSON
- 包含 JWT token 用于认证
- Token 自动续期（`createTokenRefreshScheduler`）

### 3.3 Session Runner

```typescript
// 多会话调度
async function isMultiSessionSpawnEnabled(): Promise<boolean> {
  return checkGate_CACHED_OR_BLOCKING('tengu_ccr_bridge_multi_session')
}

// --spawn 模式：同时运行多个会话
const SPAWN_SESSIONS_DEFAULT = 32

// 会话超时
const DEFAULT_SESSION_TIMEOUT_MS = 24 * 60 * 60 * 1000  // 24小时
```

### 3.4 Backoff 与容错

```typescript
const DEFAULT_BACKOFF: BackoffConfig = {
  connInitialMs: 2_000,       // 连接初始退避 2s
  connCapMs: 120_000,         // 最大退避 2min
  connGiveUpMs: 600_000,      // 放弃连接 10min
  generalInitialMs: 500,      // 通用初始退避 500ms
  generalCapMs: 30_000,       // 通用最大退避 30s
  generalGiveUpMs: 600_000,   // 通用放弃 10min
  shutdownGraceMs: 30_000,    // SIGTERM→SIGKILL 宽限期 30s
}
```

### 3.5 Sleep/Wake 检测

```typescript
function pollSleepDetectionThresholdMs(backoff: BackoffConfig): number {
  // 必须超过最大退避上限，否则正常退避延迟会误触发
  return backoff.connCapMs * 2  // 4 分钟
}
```

Bridge 通过检测 sleep/wake 来重置错误预算，避免笔记本合盖再打开后无限重试。

---

## 4. Layer 3: Forked Agent Harness

### 4.1 CacheSafeParams

> 文件：`utils/forkedAgent.ts`

Forked Agent 的核心约束——**必须与父 Agent 共享 Cache 关键参数**：

```typescript
type CacheSafeParams = {
  /** System prompt - must match parent for cache hits */
  systemPrompt: SystemPrompt
  /** User context - prepended to messages, affects cache */
  userContext: { [k: string]: string }
  /** System context - appended to system prompt, affects cache */
  systemContext: { [k: string]: string }
  /** Tool use context containing tools, model, and other options */
  toolUseContext: ToolUseContext
  /** Parent context messages for prompt cache sharing */
  forkContextMessages: Message[]
}
```

**缓存关键参数**：`system_prompt + tools + model + messages(prefix) + thinking_config`

**陷阱**：`maxOutputTokens` 会改变 `budget_tokens`（通过 claude.ts 的 clamping），而 thinking config 是缓存键的一部分。所以 Fork 的 maxOutputTokens 必须小心处理。

### 4.2 全局 CacheSafeParams Slot

```typescript
// 全局 slot，handleStopHooks 每轮写入
let lastCacheSafeParams: CacheSafeParams | null = null

// Fork 读取
export function getLastCacheSafeParams(): CacheSafeParams | null {
  return lastCacheSafeParams
}
```

**设计理由**：post-turn forks（promptSuggestion、postTurnSummary、/btw）都需要共享主循环的 Prompt Cache，但不能让每个调用方都传递参数。全局 slot 是最简单的方案。

### 4.3 用量追踪

```typescript
// Fork 完成后记录用量
logEvent({
  name: 'tengu_fork_agent_query',
  // ... 用量、时间、成功/失败
})
```

---

## 5. Layer 4: Speculation Harness（推测执行）

### 5.1 设计理念

**Speculation = 用户输入预测执行**。在用户看到 Prompt Suggestion（下一个输入建议）时，后台已经 Fork 了 Agent 开始执行：

```
时间线：
T0: 模型完成回复，生成 Prompt Suggestion
T1: 显示 Suggestion 给用户
    │
    ├── 后台：Forked Agent 开始执行 Suggestion
    │   ├── 写入 Overlay FS（不修改主文件系统）
    │   └── 最多 20 轮 / 100 条消息
    │
T2: 用户按 Tab 接受 Suggestion
    │
    ├── Overlay FS 写入 → 合并到主文件系统
    ├── Speculation 消息 → 合并到主对话
    └── 保存时间 = T2 - T0（用户感知的等待时间大幅减少）
```

### 5.2 Overlay Filesystem

```typescript
// Overlay 路径：临时目录，与主文件系统隔离
function getOverlayPath(id: string): string {
  return join(getClaudeTempDir(), 'speculation', String(process.pid), id)
}

// Speculation 写入 → Overlay
// 用户接受 → copyOverlayToMain(overlayPath)
// 用户拒绝 → safeRemoveOverlay(overlayPath)
```

**设计要点**：
- 所有文件写入先到 Overlay，不污染主工作区
- 接受时原子性地从 Overlay 拷贝到主目录
- 拒绝时递归删除 Overlay（含重试）

### 5.3 Completion Boundary

Speculation 在遇到"完成边界"时自动停止：

```typescript
type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
  | { type: 'bash'; command: string; completedAt: number }
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
```

**停止时机**：
- 模型回复完成（`complete`）
- 执行了 Bash 命令（`bash`）— 可能有副作用
- 编辑了文件（`edit`）— 已写入 Overlay
- 工具被拒绝（`denied_tool`）— 无法继续

### 5.4 Speculation 状态机

```typescript
type SpeculationState =
  | { status: 'idle' }  // 空闲
  | {
      status: 'active'  // 执行中
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean  // 是否流水线（用户还没看到 Suggestion 就开始执行）
      pipelinedSuggestion?: { ... }
    }
```

---

## 6. Layer 5: Hint Protocol（信号协议）

### 6.1 设计理念

> 文件：`utils/claudeCodeHints.ts`

Hint Protocol 是一个 **Harness-only 的带外通信通道**——CLI 工具可以向 Claude Code Harness 发送信号，但不影响模型输出。

```
外部 CLI/SDK
    │
    │  stderr: <claude-code-hint v="1" type="plugin" value="my-plugin@marketplace" />
    │
    ▼
BashTool 捕获输出
    │
    ├── extractClaudeCodeHints()
    │   ├── 解析 hint 标签
    │   ├── 从输出中剥离（模型看不到）
    │   └── 返回 { hints, stripped }
    │
    ├── stripped → 返回给模型（无 hint）
    │
    └── hints → setPendingHint()
              → React useSyncExternalStore
              → 显示安装提示给用户
```

### 6.2 协议格式

```xml
<claude-code-hint v="1" type="plugin" value="my-plugin@marketplace" />
```

| 属性 | 说明 |
|------|------|
| `v` | 协议版本（当前支持 v1） |
| `type` | hint 类型（当前只有 `plugin`） |
| `value` | hint 载荷（plugin 类型为 `name@marketplace`） |

### 6.3 安全设计

```typescript
// 1. 行锚定：防止嵌入在日志中的标签被误解析
const HINT_TAG_RE = /^[ \t]*<claude-code-hint\s+([^>]*?)\s*\/>[ \t]*$/gm

// 2. 版本/类型过滤：不认识的版本或类型直接丢弃
if (!SUPPORTED_VERSIONS.has(v)) { /* drop */ }
if (!SUPPORTED_TYPES.has(type)) { /* drop */ }

// 3. 单 slot 存储：每个会话最多显示一次
let shownThisSession = false

// 4. 剥离后输出：模型永远看不到 hint
const stripped = output.replace(HINT_TAG_RE, ...)
```

### 6.4 Harness 保证（Memdir 场景）

```typescript
// Harness guarantees the directory exists via ensureMemoryDirExists().
// Harness guarantees these directories exist so the model can write without
// first checking — the harness pre-creates them.
```

Harness 层保证记忆目录存在，模型不需要先 mkdir 再写入。

---

## 7. Layer 6: Observability Harness（可观测性）

### 7.1 Eval Harness 支持

> 文件：`services/analytics/growthbook.ts`

Feature flag 系统专门支持 eval harness（评估框架）的确定性需求：

```typescript
// Check env var overrides first (for eval harnesses)
// env-var overrides — env wins so eval harnesses remain deterministic.
// Unlike disk cache, env overrides are always available and deterministic.
```

**设计原则**：
1. **环境变量 > 服务端 > 磁盘缓存**：eval harness 通过环境变量固定 feature flag
2. **确定性**：同一个 env 设置永远得到同样的 flag 值
3. **不依赖网络**：eval 可以完全离线运行

### 7.2 Harness 权限路径

> 文件：`utils/permissions/filesystem.ts`

Harness 控制的内部路径有特殊权限：

```typescript
// 7. Allow reads from internal harness paths (session-memory, plans, tool-results)
// this subtree is harness-controlled.
```

---

## 8. 设计模式总结

| 模式 | 应用 | 价值 |
|------|------|------|
| **Dead Code Elimination** | Feature flag + Bun 编译 | 零运行时开销，代码统一 |
| **全局 CacheSafe Slot** | `lastCacheSafeParams` | 简化 Fork 参数传递 |
| **Overlay Filesystem** | Speculation 写入隔离 | 安全的推测执行 |
| **带外 Side Channel** | Hint Protocol | 模型不可见，Harness 可见 |
| **单 Slot Store** | `pendingHint` + `shownThisSession` | 每会话一次，防骚扰 |
| **Backoff + Sleep Detection** | Bridge 轮询 | 容错 + 笔记本友好 |
| **Work Secret** | Bridge 认证 | 安全的远程控制 |
| **环境变量覆盖** | Eval Harness 确定性 | 可重现的评估 |
| **Ablation Baseline** | L0 实验基线 | 科学评估功能价值 |
| **Mutable Ref 避免拷贝** | `messagesRef: { current: Message[] }` | Speculation 高性能 |

---

## 9. 可借鉴要点

### 9.1 Feature Flag DCE + 单一代码库

```
if (feature('X')) → 编译时：包含
if (feature('X')) → 编译时：移除

效果：一份代码 → 多种构建产物
  内部构建：包含 PROACTIVE, KAIROS, EXPERIMENTAL_SKILL_SEARCH
  外部构建：完全移除这些功能
```

### 9.2 Overlay Filesystem 推测执行

```typescript
// 通用 Speculation 框架
class SpeculationEngine<T> {
  private overlay: OverlayFS

  async speculate(task: T): Promise<SpeculationResult> {
    // 1. Fork 执行，写入 Overlay
    // 2. 遇到 CompletionBoundary 停止
    // 3. 返回结果 + Overlay 路径
  }

  async accept(overlayPath: string): Promise<void> {
    // 原子性合并 Overlay → 主文件系统
  }

  async reject(overlayPath: string): Promise<void> {
    // 递归删除 Overlay
  }
}
```

### 9.3 Hint Protocol 带外通信

任何需要"工具输出给模型看，但同时给 Harness 发信号"的场景：
- Plugin 推荐
- 安全告警
- 性能指标
- 环境检测

### 9.4 Ablation 实验框架

```
L0 baseline: 禁用所有增强
L1: + 记忆
L2: + 记忆 + 思考
L3: + 记忆 + 思考 + 子Agent
L4: 完整（+ 推测执行 + 后台提取）
```

通过级联的环境变量控制，科学评估每个功能的价值。

### 9.5 Bridge 远程控制模式

适用于任何需要"远程控制本地 Agent"的场景：
- Work Secret 认证（base64url JSON）
- JWT 自动续期
- 指数退避 + Sleep 检测
- 多会话调度（32 并发）
- SIGTERM → SIGKILL 优雅关闭（30s 宽限）
