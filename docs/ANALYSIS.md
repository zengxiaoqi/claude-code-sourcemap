# Claude Code 源码深度分析

> 基于 Claude Code v2.1.88 的 source map 还原源码，共 1884 个 TS/TSX 文件

## 1. 项目概览

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         main.tsx (入口)                          │
│  CLI 解析 → 认证 → 配置加载 → MCP 初始化 → REPL/Headless 启动    │
└───────────────────────────────┬─────────────────────────────────┘
                                │
          ┌─────────────────────┼──────────────────────┐
          ▼                     ▼                      ▼
   ┌──────────────┐   ┌─────────────────┐   ┌──────────────────┐
   │  REPL (交互)  │   │ QueryEngine     │   │  SDK / Headless  │
   │  Ink/React UI │   │ (查询引擎)      │   │  (非交互模式)     │
   └──────┬───────┘   └────────┬────────┘   └────────┬─────────┘
          │                    │                      │
          └────────────────────┼──────────────────────┘
                               ▼
                    ┌─────────────────────┐
                    │     query.ts         │
                    │  核心查询循环         │
                    │  API调用 → 工具执行   │
                    │  → 压缩 → 重试       │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼───────────────────────┐
          ▼                    ▼                       ▼
   ┌──────────────┐   ┌───────────────┐   ┌───────────────────┐
   │  Tool 系统    │   │ Command 系统  │   │  Coordinator 模式  │
   │  (40+ 工具)   │   │ (40+ 命令)    │   │  (多Agent协调)     │
   └──────┬───────┘   └───────────────┘   └────────┬──────────┘
          │                                         │
          ▼                                         ▼
   ┌──────────────┐   ┌───────────────┐   ┌───────────────────┐
   │ MCP 协议      │   │ Plugin 系统   │   │  Skill 系统        │
   │ (外部工具)    │   │ (可扩展插件)   │   │  (技能模块)        │
   └──────────────┘   └───────────────┘   └───────────────────┘
```

### 1.2 模块划分与文件统计

| 模块 | 文件数 | 核心职责 |
|------|--------|---------|
| `utils/` | 564 | 工具函数集（权限、模型、配置、钩子等） |
| `components/` | 389 | Ink/React 终端 UI 组件 |
| `commands/` | 207 | 斜杠命令系统 |
| `tools/` | 184 | 工具系统（Bash、文件读写、搜索等） |
| `services/` | 130 | 服务层（API、MCP、分析、压缩） |
| `hooks/` | 104 | React Hooks（权限、UI状态） |
| `ink/` | 96 | Ink 框架适配层（终端渲染引擎） |
| `bridge/` | 31 | Bridge 模式（双向通信） |
| `constants/` | 21 | 常量与配置 |
| `skills/` | 20 | 技能系统（可插拔的能力模块） |
| `cli/` | 19 | CLI 命令行定义 |
| `tasks/` | 12 | 任务管理（异步任务生命周期） |
| `types/` | 11 | 类型定义 |
| `state/` | 6 | 全局状态管理（Store 模式） |
| `coordinator/` | 3 | 多 Agent 协调模式 |
| `memdir/` | 8 | Memory 目录管理 |
| `assistant/` | 1 | 助手模式（KAIROS） |

### 1.3 技术栈

- **运行时**: Bun（TypeScript 直接运行，bundle feature flags）
- **UI 框架**: Ink (React for CLI) + 自定义终端渲染
- **语言**: TypeScript 严格模式
- **API**: Anthropic SDK (`@anthropic-ai/sdk`)
- **Schema**: Zod v4（参数验证）
- **状态管理**: 自定义 Store（函数式不可变更新）
- **构建**: Bun bundle（死代码消除 + Feature Flags）

---

## 2. 核心架构

### 2.1 启动流程

`main.tsx`（4683 行）是整个系统的入口，启动流程如下：

```
1. 启动性能追踪 (profileCheckpoint)
2. 并行预取: MDM配置、Keychain凭证、OAuth
3. CLI 参数解析 (Commander.js)
4. 认证与授权验证
5. GrowthBook Feature Flags 初始化
6. 设置/策略/迁移加载
7. MCP 服务器连接初始化
8. 技能 & 插件加载
9. 启动 REPL (交互) 或 QueryEngine (非交互)
```

**关键代码** (`main.tsx:1-25`):
```typescript
// 这些副作用必须在所有其他导入之前运行:
// 1. profileCheckpoint 在重量级模块求值前标记入口点
// 2. startMdmRawRead 触发 MDM 子进程，与后续约135ms的导入并行
// 3. startKeychainPrefetch 并行触发 macOS 钥匙串读取
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

启动时的大量并行预取是一个重要的性能优化策略。

### 2.2 核心抽象

#### QueryEngine — 查询引擎

`QueryEngine.ts` 是整个系统的核心驱动。它管理一次完整的对话会话：

```typescript
// QueryEngine 拥有对话的查询生命周期和会话状态
// 每次 submitMessage() 调用开启同一对话内的新轮次
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]       // 会话消息
  private abortController: AbortController  // 中断控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage      // 累积用量
  private readFileState: FileStateCache     // 文件读取缓存
  private discoveredSkillNames: Set<string> // 技能发现追踪
}
```

**设计亮点**：
- **单实例持久化**: 一个 QueryEngine 对应一个对话，状态跨轮次持久化
- **AsyncGenerator 模式**: `submitMessage()` 返回 AsyncGenerator，支持流式输出
- **双路径**: REPL 交互模式和 SDK/Headless 模式共用同一个引擎

#### Tool — 工具抽象

`Tool.ts` 定义了工具系统的核心接口：

```typescript
export type Tool<Input, Output, P> = {
  name: string                           // 工具名称
  aliases?: string[]                     // 别名（向后兼容）
  searchHint?: string                    // 关键词搜索提示
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  readonly inputSchema: Input            // Zod schema
  isEnabled(): boolean                   // 是否启用
  isReadOnly(input): boolean             // 是否只读
  isConcurrencySafe(input): boolean      // 是否可并发
  isDestructive?(input): boolean         // 是否破坏性操作
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  // ... UI 渲染方法 (renderToolUseMessage, renderToolResultMessage 等)
}
```

`buildTool()` 函数提供安全默认值：
```typescript
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,           // 安全默认值
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**默认值策略（Fail-Closed）**:
```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,      // 默认不可并发（安全）
  isReadOnly: () => false,             // 默认非只读（安全）
  isDestructive: () => false,
  checkPermissions: () => Promise.resolve({ behavior: 'allow', updatedInput }),
  toAutoClassifierInput: () => '',     // 默认跳过分类器
  userFacingName: () => '',            // 默认空名
}
```

#### Task — 任务抽象

`Task.ts` 定义了异步任务的类型系统：

```typescript
export type TaskType =
  | 'local_bash'        // 本地 Shell 命令
  | 'local_agent'       // 本地子 Agent
  | 'remote_agent'      // 远程 Agent
  | 'in_process_teammate' // 进程内队友
  | 'local_workflow'    // 本地工作流
  | 'monitor_mcp'       // MCP 监控
  | 'dream'             // 梦境模式

export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

每个任务有唯一 ID（前缀 + 8位随机字符），如 `b7x9k2m1p` 表示 bash 任务。

### 2.3 模块间关系

```
AppState (全局状态) ←── Store (不可变更新模式)
     ↑
     ├── REPL / QueryEngine (读写状态)
     ├── Tools (读取 tools/mcp 状态)
     ├── Commands (读写 settings)
     └── Coordinator (管理 tasks/agents)

query.ts (核心循环)
     ├── 调用 Anthropic API
     ├── 通过 toolOrchestration.ts 执行工具
     ├── 通过 compact.ts 压缩上下文
     └── 生成消息流 (AsyncGenerator)
```

---

## 3. 模块详解

### 3.1 工具系统 (tools/)

Claude Code 的工具系统包含 40+ 工具，分为以下类别：

**核心工具**:
| 工具 | 文件 | 功能 |
|------|------|------|
| `BashTool` | `BashTool/` | Shell 命令执行 |
| `FileReadTool` | `FileReadTool/` | 文件读取 |
| `FileEditTool` | `FileEditTool/` | 文件编辑 |
| `FileWriteTool` | `FileWriteTool/` | 文件写入 |
| `GlobTool` | `GlobTool/` | 文件搜索（glob） |
| `GrepTool` | `GrepTool/` | 内容搜索（grep） |
| `WebSearchTool` | `WebSearchTool/` | 网络搜索 |
| `WebFetchTool` | `WebFetchTool/` | 网页抓取 |

**Agent/任务工具**:
| 工具 | 功能 |
|------|------|
| `AgentTool` | 启动子 Agent（worker） |
| `TaskOutputTool` | 获取任务输出 |
| `TaskStopTool` | 停止任务 |
| `TaskCreateTool/GetTool/UpdateTool/ListTool` | 任务 CRUD |
| `TeamCreateTool/TeamDeleteTool` | 团队管理 |
| `SendMessageTool` | 向 Agent 发消息 |

**规划与模式工具**:
| 工具 | 功能 |
|------|------|
| `EnterPlanModeTool` | 进入规划模式 |
| `ExitPlanModeTool` | 退出规划模式 |
| `EnterWorktreeTool/ExitWorktreeTool` | Git worktree 管理 |
| `ToolSearchTool` | 搜索可用工具（延迟加载） |

**MCP 工具**: 通过 MCP 协议动态注册的外部工具

**工具注册流程** (`tools.ts`):

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, TaskOutputTool, BashTool, GlobTool, GrepTool,
    ExitPlanModeV2Tool, FileReadTool, FileEditTool, FileWriteTool,
    // ... 条件注册
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    ...(isTodoV2Enabled() ? [TaskCreateTool, ...] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

**工具过滤与组装** (`tools.ts:assembleToolPool`):
```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  // 按名称排序 + 去重（内置工具优先）
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

### 3.2 工具执行与并发控制

**`toolOrchestration.ts`** — 工具编排器：

核心设计是将工具调用分为**可并发**和**串行**两类：

```typescript
function partitionToolCalls(toolUseMessages, toolUseContext): Batch[] {
  // 将工具调用分区：
  // 1. 连续的只读工具 → 并发执行
  // 2. 非只读工具 → 串行执行
}

async function* runTools(toolUseMessages, ...) {
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(...)) {
    if (isConcurrencySafe) {
      // 只读工具并发执行
      yield* runToolsConcurrently(blocks, ...)
    } else {
      // 写工具串行执行
      yield* runToolsSerially(blocks, ...)
    }
  }
}
```

**`StreamingToolExecutor.ts`** — 流式工具执行器：

支持工具在流式响应中**边接收边执行**：
```typescript
export class StreamingToolExecutor {
  // 工具边流边执行，保持输出顺序
  addTool(block: ToolUseBlock, assistantMessage: AssistantMessage): void {
    // 立即检查是否可执行
    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      await this.executeTool(tool)
    }
  }

  // 并发规则：
  // - 只读工具可以与其他只读工具并发
  // - 写工具必须独占执行
  private canExecuteTool(isConcurrencySafe: boolean): boolean {
    const executing = this.tools.filter(t => t.status === 'executing')
    return executing.length === 0 ||
      (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  }
}
```

### 3.3 命令系统 (commands/)

40+ 斜杠命令，每个命令独立目录/文件。命令系统通过 `commands.ts` 集中注册：

**核心命令分类**:
- **会话管理**: `/resume`, `/session`, `/clear`, `/compact`
- **文件操作**: `/diff`, `/copy`, `/files`
- **Git 操作**: `/commit`, `/review`, `/commit-push-pr`, `/pr_comments`
- **配置管理**: `/config`, `/mcp`, `/doctor`, `/init`
- **模型控制**: `/model` (通过 fast/effort), `/cost`
- **安全**: `/permissions`, `/security-review`
- **Agent**: `/agents`, `/tasks`, `/workflows`

**Feature Flag 控制**: 许多命令通过 `feature()` 或环境变量条件注册：
```typescript
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default : null
const briefCommand = feature('KAIROS') || feature('KAIROS_BRIEF')
  ? require('./commands/brief.js').default : null
```

### 3.4 多 Agent 协调模式 (coordinator/)

这是 Claude Code 最核心的设计之一。Coordinator 模式实现了一个 **协调者-工作者** 架构：

**协调者（Coordinator）**:
- 通过 `AgentTool` 启动 Worker
- 通过 `SendMessageTool` 继续已存在的 Worker
- 通过 `TaskStopTool` 停止 Worker
- 综合各 Worker 的研究结果，编写精确的实现规格

**工作者（Worker）**:
- 独立运行，拥有自己的上下文
- 可访问的工具集受限（`ASYNC_AGENT_ALLOWED_TOOLS`）
- 支持 MCP 工具和项目技能
- 完成后通过 `<task-notification>` XML 通知协调者

**核心系统提示词** (`coordinatorMode.ts`):

```
你是一个协调者。你的职责是：
1. 帮助用户实现目标
2. 指导 worker 进行研究、实现和验证
3. 综合结果并与用户沟通
4. 能直接回答的问题不要委派
```

**任务工作流**:

| 阶段 | 执行者 | 目的 |
|------|--------|------|
| 研究 | Workers（并行） | 调查代码库、定位文件、理解问题 |
| 综合 | **协调者** | 阅读发现、理解问题、编写实现规格 |
| 实现 | Workers | 按规格进行代码修改 |
| 验证 | Workers | 测试修改是否生效 |

**关键设计：始终综合 — 你最重要的工作**

```typescript
// 反模式 — 懒惰委派
AgentTool({ prompt: "基于你的发现，修复 auth bug" })

// 正确 — 综合后的规格
AgentTool({ prompt: "修复 src/auth/validate.ts:42 的空指针。
  Session 的 user 字段在 session 过期但 token 仍缓存时为 undefined。
  在访问 user.id 前添加空值检查 — 如果为 null，返回 401 并附带
  'Session expired'。提交并报告 hash。" })
```

### 3.5 服务层 (services/)

**API 服务** (`services/api/`):
- Claude API 调用与重试
- 流式响应处理
- Token 计数与预算管理

**MCP 协议** (`services/mcp/`):
- MCP 客户端管理（`MCPConnectionManager.tsx`）
- 服务器连接、认证、资源配置
- Elicitation 处理（MCP 工具向用户请求信息）
- 多传输层支持（InProcess、SdkControl）

**上下文压缩** (`services/compact/`):
- 自动压缩（`autoCompact.ts`）
- 对话历史压缩（`compact.ts`）
- Snip 压缩（`snipCompact.ts`）— 实验性

**分析服务** (`services/analytics/`):
- GrowthBook Feature Flags 集成
- 事件追踪与遥测

### 3.6 技能系统 (skills/)

技能系统通过目录结构组织可插拔的能力模块：

```
skills/
  bundled/         # 内置技能
  bundledSkills.ts # 技能注册
  loadSkillsDir.ts # 从目录加载技能
  mcpSkillBuilders.ts # MCP 技能构建器
```

技能与命令系统协同工作 — `SkillTool` 允许 Agent 调用技能。

### 3.7 状态管理 (state/)

Claude Code 使用**不可变 Store 模式**：

**Store** (`state/store.ts`):
```typescript
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(initialState: T, onChange?: OnChange<T>): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等则跳过
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: (listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

**AppState** (`state/AppStateStore.ts`) 是全局状态的巨型类型（~300行），包含：
- `toolPermissionContext` — 工具权限上下文
- `tasks` — 异步任务状态
- `mcp` — MCP 连接/工具/资源
- `plugins` — 插件状态
- `agentDefinitions` — Agent 定义
- `speculation` — 推测执行状态
- `teamContext` — 团队协作上下文
- 以及更多...

**DeepImmutable** 包装确保状态的不可变性，只有 `tasks` 因包含函数类型而豁免。

### 3.8 权限系统

权限是多层级的：

1. **工具级别**: `checkPermissions()` + `validateInput()`
2. **规则级别**: `alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules`
3. **模式级别**: `PermissionMode` (`default` / `plan` / `auto` / `bypassPermissions`)
4. **分类器**: 自动模式下的 AI 分类器判断操作安全性
5. **交互确认**: UI 弹窗让用户确认

**权限检查流程** (`useCanUseTool.tsx`):
```
工具调用 → hasPermissionsToUseTool()
         → allow? → 直接通过
         → deny?  → 记录拒绝
         → ask?   → 交互式确认
                    ├── coordinator handler (协调者模式)
                    ├── swarm worker handler (集群模式)
                    └── interactive handler (交互模式)
```

### 3.9 上下文压缩 (compact)

当对话历史过长时，自动压缩上下文：

```typescript
// compact.ts 中的压缩流程：
// 1. 执行 pre-compact 钩子
// 2. 分析上下文（token 统计）
// 3. 使用 forkedAgent 生成摘要
// 4. 创建 compact boundary 消息
// 5. 执行 post-compact 钩子
// 6. 替换历史消息为压缩版本
```

### 3.10 Feature Flags 系统

Claude Code 使用 Bun 的 `feature()` 进行编译时死代码消除：

```typescript
// 编译时特性开关
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null

// 运行时特性开关
const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default
  : null
```

这种设计让同一份代码可以编译出不同的版本（Ant 内部版 vs 开源版），未使用的特性在编译时被完全消除。

---

## 4. 设计模式提炼

### 4.1 工具系统的抽象与扩展 — Plugin Architecture 模式

**模式**: 定义统一的工具接口，通过 `buildTool()` 提供安全默认值，支持 MCP 动态扩展。

```typescript
// Tool.ts — 统一接口
export type Tool<Input, Output, P> = {
  call(args, context, canUseTool, parentMessage, onProgress): Promise<ToolResult<Output>>
  isEnabled(): boolean
  isReadOnly(input): boolean
  isConcurrencySafe(input): boolean
  checkPermissions(input, context): Promise<PermissionResult>
  // ...
}

// buildTool() — 安全默认值（Fail-Closed）
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,   // isEnabled: true, isConcurrencySafe: false, ...
    ...def,
  }
}

// tools.ts — 统一注册与过滤
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,  // MCP 动态工具
): Tools {
  const builtIn = getTools(permissionContext)
  const allowed = filterToolsByDenyRules(mcpTools, permissionContext)
  return uniqBy([...builtIn].sort(byName).concat(allowed.sort(byName)), 'name')
}
```

**关键设计决策**:
- **Fail-Closed 默认值**: `isConcurrencySafe` 默认 `false`，`isReadOnly` 默认 `false`，确保新工具不会意外获得危险权限
- **MCP 扩展点**: MCP 工具通过相同的 `Tool` 接口注册，内置工具和外部工具对系统完全透明
- **按需加载**: `ToolSearchTool` 支持工具延迟加载，减少 prompt token 消耗

### 4.2 异步生成器驱动的主循环 — Generator Pattern

**模式**: 整个查询流程由 AsyncGenerator 驱动，实现流式处理和早期产出。

```typescript
// QueryEngine.ts
async *submitMessage(prompt, options): AsyncGenerator<SDKMessage, void, unknown> {
  // 1. 处理用户输入
  const { messages, shouldQuery } = await processUserInput({ input: prompt, ... })
  
  // 2. 将消息推入历史
  this.mutableMessages.push(...messages)
  
  // 3. 进入核心查询循环
  for await (const message of query({ messages, systemPrompt, ... })) {
    // 4. 按类型处理并 yield
    switch (message.type) {
      case 'assistant': yield* normalizeMessage(message); break
      case 'user':      yield* normalizeMessage(message); break
      case 'progress':  yield* normalizeMessage(message); break
      // ...
    }
    
    // 5. 检查预算限制
    if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
      yield { type: 'result', subtype: 'error_max_budget_usd', ... }
      return
    }
  }
  
  // 6. 产出最终结果
  yield { type: 'result', subtype: 'success', result: textResult, ... }
}
```

**为什么用 AsyncGenerator 而不是回调/Promise**:
- 流式输出：每产出一条消息就立即返回给调用者
- 背压控制：调用者决定何时拉取下一条消息
- 可取消性：调用者可以随时停止迭代
- 组合性：`yield*` 委托子生成器

### 4.3 不可变 Store — Functional Update Pattern

**模式**: 类似 Redux 的不可变状态更新，但极简实现。

```typescript
// store.ts — 极简 Store
export function createStore<T>(initialState: T, onChange?): Store<T> {
  let state = initialState
  return {
    getState: () => state,
    setState: (updater: (prev: T) => T) => {
      const next = updater(state)
      if (Object.is(next, state)) return  // 引用相等优化
      state = next
      // 通知监听器
    },
    subscribe: (listener) => () => listeners.delete(listener)
  }
}

// 使用方式 — 函数式更新
store.setState(prev => ({
  ...prev,
  toolPermissionContext: {
    ...prev.toolPermissionContext,
    alwaysAllowRules: { ...prev.toolPermissionContext.alwaysAllowRules, command: allowedTools }
  }
}))
```

**优势**:
- 无中间件复杂性，纯函数式更新
- 引用相等检查避免不必要的重渲染
- `DeepImmutable<T>` 类型包装确保编译时不可变性

### 4.4 多 Agent 协调 — Orchestrator-Worker Pattern

**模式**: 协调者通过结构化提示词控制工作者，工作者自治执行。

```
用户 ──→ Coordinator ──→ Worker A (研究)
              │    ───→ Worker B (研究)
              │
              ←── task-notification (Worker A 完成)
              │
              ──→ Worker A (继续：实现) 或 Worker C (新：实现)
              │
              ←── task-notification (实现完成)
              │
              ──→ Worker D (验证)
              │
              ←── task-notification (验证完成)
              │
         综合结果 ──→ 用户
```

**关键设计**:
1. **Worker 看不到协调者的对话** — 每次 prompt 必须自包含
2. **协调者必须综合** — 不能把理解工作推给 Worker
3. **并发是超能力** — 独立的研究任务并行执行
4. **Continue vs Spawn** — 根据上下文重叠度决定复用还是新建

### 4.5 权限分层 — Defense in Depth Pattern

```
Level 1: validateInput() — 输入验证（路径检查等）
Level 2: checkPermissions() — 工具特定权限逻辑
Level 3: hasPermissionsToUseTool() — 通用规则匹配
Level 4: PermissionMode — 模式级别控制
Level 5: AI Classifier — 自动模式下的智能判断
Level 6: Interactive Dialog — 用户确认弹窗
Level 7: Hooks (PreToolUse/PostToolUse) — 外部钩子
```

### 4.6 流式工具执行 — Streaming Executor Pattern

```typescript
// StreamingToolExecutor.ts
class StreamingToolExecutor {
  // 工具边流边执行
  addTool(block, assistantMessage) {
    // 立即检查是否可执行
    if (this.canExecuteTool(tool.isConcurrencySafe)) {
      this.executeTool(tool)  // 开始执行
    }
  }
  
  // 并发控制：只读并行 + 写操作串行
  canExecuteTool(isConcurrencySafe) {
    const executing = this.tools.filter(t => t.status === 'executing')
    return executing.length === 0 ||
      (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
  }
}
```

### 4.7 Feature Flags + Dead Code Elimination

```typescript
// 编译时条件导入 — 未启用的特性完全不包含在产物中
const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null

// 使用时安全检查
if (feature('COORDINATOR_MODE') && coordinatorModeModule?.isCoordinatorMode()) {
  // ...
}
```

### 4.8 上下文压缩策略 — Adaptive Compaction

```typescript
// 自动压缩触发条件
export function isAutoCompactEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_compact') !== 'off'
}

// 压缩流程
// 1. token 计数超过阈值 → 触发压缩
// 2. 使用 forkedAgent 生成对话摘要
// 3. 创建 compact boundary 标记
// 4. 替换历史为压缩版本
```

---

## 5. 可借鉴的设计思路

### 5.1 Agent 项目中的工具系统设计

**借鉴方案**: 采用 Claude Code 的 `Tool` 接口 + `buildTool()` 模式。

```typescript
// 1. 定义统一工具接口
interface Tool<I, O> {
  name: string
  schema: ZodSchema<I>
  execute(input: I, ctx: Context): Promise<O>
  // 能力声明（用于调度决策）
  isReadOnly(input: I): boolean
  isConcurrencySafe(input: I): boolean
  isDestructive(input: I): boolean
}

// 2. 安全默认值
function createTool<I, O>(def: Partial<Tool<I, O>> & Pick<Tool<I, O>, 'name' | 'execute' | 'schema'>): Tool<I, O> {
  return {
    isReadOnly: () => false,
    isConcurrencySafe: () => false,
    isDestructive: () => false,
    ...def,
  }
}

// 3. 工具注册表支持动态扩展
class ToolRegistry {
  private tools = new Map<string, Tool>()
  register(tool: Tool) { this.tools.set(tool.name, tool) }
  get(name: string) { return this.tools.get(name) }
  getAll(context: PermissionContext): Tool[] {
    return [...this.tools.values()].filter(t => this.isAllowed(t, context))
  }
}
```

### 5.2 多 Agent 协调架构

**借鉴方案**: Coordinator-Worker 模式 + 结构化提示词。

关键原则：
1. **协调者综合，不委派理解** — Worker 的 prompt 必须包含协调者综合后的具体信息
2. **并行是超能力** — 研究阶段可以并发多个 Worker
3. **Continue vs Spawn 策略** — 根据上下文重叠度选择
4. **Worker 通过通知机制回报** — 异步通知而非轮询

```typescript
class Coordinator {
  async handleUserMessage(message: string) {
    // 阶段 1: 并行研究
    const [research1, research2] = await Promise.all([
      this.spawnWorker({ task: '研究代码库中的 auth 模块...' }),
      this.spawnWorker({ task: '查找 auth 相关测试...' }),
    ])
    
    // 阶段 2: 协调者综合
    const spec = this.synthesize(research1, research2)
    
    // 阶段 3: 实现Worker
    const impl = await this.spawnWorker({ task: spec })
    
    // 阶段 4: 验证Worker（独立验证，不复用上下文）
    const verify = await this.spawnWorker({ task: `验证 ${impl.summary}...` })
  }
}
```

### 5.3 流式 Agent 执行引擎

**借鉴方案**: AsyncGenerator 驱动的查询循环。

```typescript
class AgentEngine {
  async *runTurn(userMessage: string): AsyncGenerator<AgentEvent> {
    // 1. 处理用户输入
    yield { type: 'processing', stage: 'input' }
    
    // 2. 构建 prompt + 发起 API 调用
    const stream = this.client.messages.stream({
      messages: this.buildMessages(userMessage),
      tools: this.getToolDefinitions(),
    })
    
    // 3. 流式处理响应
    for await (const event of stream) {
      if (event.type === 'content_block_start' && event.content_block.type === 'tool_use') {
        // 工具边接收边执行
        yield { type: 'tool_start', tool: event.content_block.name }
      }
      
      if (event.type === 'message_stop') {
        // 检查是否需要继续
        if (this.hasToolResults()) {
          yield* this.executeTools()  // 递归
        }
      }
      
      yield { type: 'stream_event', event }
    }
    
    // 4. 检查预算
    if (this.exceedsBudget()) {
      yield { type: 'budget_exceeded' }
      return
    }
    
    // 5. 最终结果
    yield { type: 'result', text: this.extractText() }
  }
}
```

### 5.4 终端 UI 架构

**借鉴方案**: Ink/React for CLI。

Claude Code 的 UI 组件结构（389个文件）证明了 Ink 模式的可扩展性：
- 组件化渲染：每个工具的显示逻辑独立封装在 `renderToolUseMessage` / `renderToolResultMessage` 中
- 状态驱动：AppState 变化自动触发重渲染
- 响应式布局：适配终端尺寸变化

### 5.5 错误处理与恢复策略

**借鉴方案**: 多层错误恢复。

```typescript
// 1. API 层面：自动重试
const result = await withRetry(apiCall, {
  maxRetries: 5,
  shouldRetry: (err) => isRetryableAPIError(err),
  onRetry: (attempt) => logEvent('api_retry', { attempt }),
})

// 2. 工具层面：错误隔离
try {
  const result = await tool.call(input, context)
} catch (err) {
  // 工具错误不影响主循环，返回错误消息让模型决定下一步
  return createUserMessage({
    content: [{ type: 'tool_result', content: `Error: ${err.message}`, is_error: true, tool_use_id }]
  })
}

// 3. 会话层面：transcript 持久化
// 即使进程崩溃，也能通过 --resume 恢复对话
await recordTranscript(messages)
```

### 5.6 可扩展的插件/技能系统

**借鉴方案**: 基于目录的技能发现 + MCP 协议。

```typescript
// 技能发现：扫描特定目录
const skills = await loadSkillsDir(getCwd())

// 插件加载：支持远程安装 + 本地缓存
const { enabled } = await loadAllPluginsCacheOnly()

// MCP 扩展：标准化的外部工具协议
const mcpTools = await getMcpToolsCommandsAndResources(mcpClients)
```

---

## 6. 总结与建议

### 6.1 架构亮点

1. **工具系统的 Fail-Closed 设计** — 所有默认值偏向安全，新工具必须显式声明能力
2. **AsyncGenerator 驱动的主循环** — 天然支持流式、背压、取消
3. **Coordinator-Worker 多 Agent 架构** — 协调者综合理解、Worker 自治执行，实现了真正的任务并行
4. **Feature Flags + 死代码消除** — 一份代码编译多版本，零运行时开销
5. **流式工具执行器** — 边接收边执行，最大化并发效率
6. **不可变 Store** — 极简但功能完整的状态管理

### 6.2 可改进的方向

1. **main.tsx 过于庞大**（4683行）— 可以将启动流程拆分为独立的阶段模块
2. **AppState 过于庞大**（~300行类型定义）— 可以按领域拆分为多个 Store
3. **条件注册散落各处** — Feature Flag 的条件导入模式重复出现，可以抽象为声明式注册
4. **错误处理可统一** — 目前错误处理分散在各层，可以引入统一的 Error Boundary

### 6.3 对构建 Agent 项目的建议

1. **从 Tool 接口开始设计** — 统一的工具抽象是 Agent 系统的基石
2. **AsyncGenerator 作为核心执行模型** — 流式、可取消、可组合
3. **安全默认值优先** — 永远假设最坏情况，显式授权而非显式拒绝
4. **协调者综合，不委派理解** — 这是多 Agent 系统成功的关键
5. **Feature Flags 驱动开发** — 让同一份代码支持不同配置
6. **Transcript 持久化** — 每一步都要保存，崩溃可恢复

---

> 本文档基于 Claude Code v2.1.88 source map 还原源码分析，文件路径均相对于 `restored-src/src/`。
