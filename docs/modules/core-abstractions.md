# 核心抽象层（Core Abstractions）

## 模块概览

| 指标 | 值 |
|------|-----|
| 文件数 | 12 |
| 总代码行数 | ~11,259 |
| 核心职责 | 定义查询引擎生命周期、任务类型系统、工具抽象接口、命令注册、成本追踪、会话管理 |
| 关键文件 | `QueryEngine.ts`, `main.tsx`, `query.ts`, `Tool.ts`, `tools.ts`, `commands.ts` |

### 关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.tsx` | 4,683 | 启动入口、CLI 解析、初始化编排 |
| `query.ts` | 1,729 | 核心查询循环（API 调用 + 工具执行） |
| `QueryEngine.ts` | 1,295 | 查询引擎类（SDK/Headless 路径） |
| `Tool.ts` | 792 | 工具抽象接口定义 |
| `commands.ts` | 754 | 命令注册表（50+ 斜杠命令） |
| `tools.ts` | 389 | 工具注册表（全局工具池） |
| `cost-tracker.ts` | 323 | 成本/用量追踪 |
| `history.ts` | 464 | 命令历史管理 |
| `setup.ts` | 477 | 初始化设置流程 |
| `context.ts` | 189 | 上下文构建（Git 状态、系统提示） |
| `Task.ts` | 125 | 任务类型系统 |
| `tasks.ts` | 39 | 任务管理器（工厂模式） |

---

## 1. QueryEngine — 查询引擎生命周期

**文件路径**: `src/QueryEngine.ts` (1,295 行)

### 架构设计

```
QueryEngine (per-conversation)
├── constructor(config: QueryEngineConfig)
├── submitMessage(prompt, options?) → AsyncGenerator<SDKMessage>
│   ├── processUserInput() — 处理用户输入 + 斜杠命令
│   ├── recordTranscript() — 持久化到磁盘
│   ├── query() — 进入核心查询循环
│   │   ├── normalizeMessagesForAPI() — 消息标准化
│   │   ├── callModel() — 流式 API 调用
│   │   ├── runTools() — 工具执行
│   │   └── yield messages — 逐步向调用方返回
│   └── handleOrphanedPermission() — 权限孤儿处理
├── mutableMessages: Message[] — 会话消息存储
├── abortController — 中断控制
└── totalUsage — 累积用量追踪
```

### 核心代码

```typescript
// QueryEngineConfig 定义 — 完整的查询配置
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  // ... 更多配置
}
```

**关键设计决策**:
- `submitMessage()` 返回 `AsyncGenerator<SDKMessage>`，支持流式迭代
- `mutableMessages` 在多次 `submitMessage()` 间共享，实现会话持久化
- `snipReplay` 回调用于 HISTORY_SNIP 特性，将边界压缩逻辑注入引擎
- 通过 `feature('COORDINATOR_MODE')` 条件导入协调模式，实现死代码消除

### 设计优缺点

**优点**:
- 将 SDK/Headless 路径从 REPL 中抽离，职责单一
- AsyncGenerator 模式天然支持流式输出
- 配置对象集中管理，易于测试

**缺点**:
- `QueryEngineConfig` 参数过多（20+ 字段），可进一步分组
- `submitMessage()` 内部逻辑仍然较重（~700 行），可拆分子方法

---

## 2. Task — 任务类型系统

**文件路径**: `src/Task.ts` (125 行)

### 架构设计

```
TaskType (枚举)
├── local_bash      — 本地 Shell 命令 (前缀 'b')
├── local_agent     — 本地 Agent 子进程 (前缀 'a')
├── remote_agent    — 远程 Agent (前缀 'r')
├── in_process_teammate — 进程内队友 (前缀 't')
├── local_workflow  — 本地工作流 (前缀 'w')
├── monitor_mcp     — MCP 监控 (前缀 'm')
└── dream           — Dream 任务 (前缀 'd')

TaskStatus
├── pending → running → completed
│                    → failed
│                    → killed

Task (接口)
├── name: string
├── type: TaskType
└── kill(taskId, setAppState) → Promise<void>
```

### 核心代码

```typescript
export type TaskHandle = {
  taskId: string
  cleanup?: () => void
}

export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}

// Task ID 生成 — 前缀 + 8位随机字符，36^8 ≈ 2.8万亿组合
export function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type)
  const bytes = randomBytes(8)
  let id = prefix
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length]
  }
  return id
}
```

**设计特点**:
- Task ID 采用前缀 + 随机字符的方案，通过前缀即可识别任务类型
- `isTerminalTaskStatus()` 判断终态，用于防止向已结束任务注入消息
- `Task` 接口仅保留 `kill` 方法（多态分发的唯一入口）

---

## 3. Tool — 工具抽象接口

**文件路径**: `src/Tool.ts` (792 行)

### 架构设计

```
Tool<Input, Output, Progress>
├── name: string                    — 工具名称
├── aliases?: string[]              — 别名（向后兼容）
├── inputSchema: AnyObject          — Zod 输入验证
├── call(args, context, ...)        — 执行工具
├── description(input, options)     — 生成描述文本
├── checkPermissions(input, ctx)    — 权限检查
├── validateInput(input, ctx)       — 输入验证
├── isEnabled()                     — 是否启用
├── isReadOnly(input)               — 是否只读
├── isDestructive?(input)           — 是否破坏性
├── isConcurrencySafe(input)        — 是否并发安全
├── interruptBehavior?()            — 中断行为 (cancel | block)
├── prompt(options)                 — 生成提示词
├── mapToolResultToToolResultBlockParam() — 结果序列化
├── renderToolResultMessage?()      — UI 渲染
├── shouldDefer?: boolean           — 延迟加载（ToolSearch）
├── alwaysLoad?: boolean            — 始终加载
├── mcpInfo?                        — MCP 工具信息
└── maxResultSizeChars              — 结果大小限制

ToolUseContext (执行上下文)
├── options (工具列表、命令、模型、MCP等)
├── messages: Message[]
├── abortController
├── getAppState / setAppState
├── readFileState: FileStateCache
└── 30+ 可选回调字段

ToolResult<T>
├── data: T
├── newMessages?
└── contextModifier?
```

### 核心代码

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  readonly name: string
  readonly inputSchema: Input
  maxResultSizeChars: number
  
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  
  description(input, options): Promise<string>
  checkPermissions(input, context): Promise<PermissionResult>
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  prompt(options): Promise<string>
  // ...
}
```

**关键设计**:
- **ToolSearch 机制**: `shouldDefer` 允许工具延迟加载，通过 `ToolSearchTool` 按需激活
- **并发安全标记**: `isConcurrencySafe()` 决定工具是否可并行执行
- **权限分层**: `validateInput` → `checkPermissions` → `canUseTool` 三级检查
- **结果大小控制**: `maxResultSizeChars` 超限时自动持久化到磁盘，返回文件路径

---

## 4. query.ts — 核心查询循环

**文件路径**: `src/query.ts` (1,729 行)

### 架构设计

```
query(params: QueryParams) → AsyncGenerator<Message | StreamEvent>

查询循环:
┌─────────────────────────────────┐
│  1. 构建系统提示                  │
│  2. Token 预算检查               │
│  3. while (attemptWithFallback) │
│     ├── for await (callModel)   │
│     │   ├── yield assistant消息  │
│     │   ├── 检查 max_output_tokens│
│     │   └── 收集 tool_use 块     │
│     ├── runTools(toolUseBlocks)  │
│     │   └── yield tool_result   │
│     ├── 自动压缩检查             │
│     └── 模型回退处理             │
│  4. 执行 stop hooks             │
│  5. 返回终止原因                 │
└─────────────────────────────────┘
```

### 关键代码片段

```typescript
// 查询参数定义
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  taskBudget?: { total: number }
  deps?: QueryDeps
}

// 流式 API 调用 + 工具执行的完整循环
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model: currentModel, ... },
})) {
  // 处理流式消息...
  // 检查 prompt-too-long 恢复
  // 检查 max_output_tokens 恢复
}
```

**关键特性**:
- **Thinking 规则**: thinking 块必须在完整的 trajectory 中保持（thinking → tool_use → tool_result → next assistant）
- **模型回退**: 主模型失败时自动切换到 fallbackModel
- **Token 预算**: `checkTokenBudget()` 控制 turn 级别的 token 消耗
- **自动压缩**: `reactiveCompact` 和 `contextCollapse` 两套溢出恢复机制
- **流式丢弃**: streaming fallback 时通过 tombstone 消息清除已展示的部分消息

---

## 5. main.tsx — 启动流程

**文件路径**: `src/main.tsx` (4,683 行)

### 启动时序

```
main.tsx 启动流程:
1. profileCheckpoint('main_tsx_entry')     — 性能追踪起点
2. startMdmRawRead()                       — 并行启动 MDM 读取
3. startKeychainPrefetch()                 — 并行预取 Keychain
4. 解析 CLI 参数 (Commander.js)
5. 初始化 GrowthBook (A/B 测试)
6. 认证检查 (OAuth / API Key)
7. 策略限制加载 (policyLimits)
8. 远程设置加载 (remoteManagedSettings)
9. MCP 配置加载
10. setup() 调用 — 初始化环境
11. launchRepl() 或 直接执行
```

### 关键代码

```typescript
// 启动时的并行预取策略
import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

// MDM 子进程并行运行（plutil/reg query）
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();

// macOS Keychain 预读（OAuth + API Key 并行）
import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

**设计特点**:
- 4,683 行的单文件入口，承载了 CLI 定义 + 环境初始化 + 会话恢复 + 多种启动模式
- 大量使用 `feature()` 进行条件导入，实现死代码消除
- `USER_TYPE === 'ant'` 条件用于 Ant 内部构建 vs 外部构建的分支
- 迁移系统（migrations/）确保配置格式升级平滑

---

## 6. context.ts — 上下文构建

**文件路径**: `src/context.ts` (189 行)

### 职责
构建注入到系统提示中的用户上下文和系统上下文。

```typescript
// Git 状态注入
export const getGitStatus = memoize(async (): Promise<string | null> => {
  const [branch, mainBranch, status, log, userName] = await Promise.all([
    getBranch(),
    getDefaultBranch(),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'status', '--short']),
    execFileNoThrow(gitExe(), ['--no-optional-locks', 'log', '--oneline', '-n', '5']),
    execFileNoThrow(gitExe(), ['config', 'user.name']),
  ])
  // 最多 2000 字符，超出截断
})
```

---

## 7. tools.ts — 工具注册表

**文件路径**: `src/tools.ts` (389 行)

### 架构

```
工具注册层次:
getAllBaseTools() → Tool[] (全量基础工具)
  ↓
getTools(permissionContext) → Tools (按权限过滤)
  ↓
assembleToolPool(permissionContext, mcpTools) → Tools (合并 MCP 工具)
```

```typescript
// 工具列表构建 — 条件编译
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool, FileEditTool, FileWriteTool,
    // ... 更多工具
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
  ]
}
```

**设计特点**:
- **Simple 模式**: `CLAUDE_CODE_SIMPLE` 仅暴露 Bash/Read/Edit 三个工具
- **REPL 模式**: 隐藏底层工具，通过 VM 封装
- **Coordinator 模式**: 追加 Agent/SendMessage/TaskStop 工具
- `assembleToolPool()` 是工具池组装的唯一真相源

---

## 8. commands.ts — 命令注册表

**文件路径**: `src/commands.ts` (754 行)

注册 50+ 斜杠命令，包括：
- `/commit`, `/review`, `/pr-comments` — Git 工作流
- `/compact`, `/memory`, `/context` — 会话管理
- `/mcp`, `/config`, `/permissions` — 配置管理
- `/resume`, `/session`, `/share` — 会话操作
- 条件命令：`/assistant` (KAIROS)、`/bridge` (BRIDGE_MODE)、`/voice` (VOICE_MODE)

---

## 9. cost-tracker.ts — 成本追踪

**文件路径**: `src/cost-tracker.ts` (323 行)

```typescript
// 成本持久化到项目配置
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [
        getCanonicalName(model),
        { inputTokens, outputTokens, cacheReadInputTokens, ... }
      ]),
    ),
    lastSessionId: getSessionId(),
  }))
}
```

---

## 10. history.ts — 命令历史管理

**文件路径**: `src/history.ts` (464 行)

- 基于 JSONL 的持久化历史（`~/.claude/history.jsonl`）
- 支持粘贴内容的外部存储（hash 引用 → pasteStore）
- 当前会话优先排序，项目过滤，最多 100 条
- 支持时间戳去重的 Ctrl+R 搜索

---

## 11. setup.ts — 初始化设置

**文件路径**: `src/setup.ts` (477 行)

启动时执行的关键初始化：
- Node.js 版本检查（≥18）
- UDS 消息服务器启动
- Teammate 快照捕获
- 终端备份恢复（iTerm2 / Terminal.app）
- Worktree 创建（可选）
- Hooks 配置快照
- 命令注册（`getCommands()`）

---

## 12. tasks.ts — 任务管理器

**文件路径**: `src/tasks.ts` (39 行)

```typescript
export function getAllTasks(): Task[] {
  const tasks: Task[] = [
    LocalShellTask,
    LocalAgentTask,
    RemoteAgentTask,
    DreamTask,
  ]
  if (LocalWorkflowTask) tasks.push(LocalWorkflowTask)
  if (MonitorMcpTask) tasks.push(MonitorMcpTask)
  return tasks
}

export function getTaskByType(type: TaskType): Task | undefined {
  return getAllTasks().find(t => t.type === type)
}
```

工厂模式 + 条件加载，与 `tools.ts` 保持一致的模式。

---

## 整体架构图

```
                    ┌─────────────┐
                    │  main.tsx   │ ← CLI 解析 + 启动编排
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   setup.ts  │ ← 环境初始化
                    └──────┬──────┘
                           │
              ┌────────────▼────────────┐
              │  QueryEngine / REPL     │
              │  (会话管理 + 消息循环)    │
              └────────────┬────────────┘
                           │
                    ┌──────▼──────┐
                    │  query.ts   │ ← 核心查询循环
                    │  (API+工具)  │
                    └──────┬──────┘
                           │
              ┌────────────▼────────────┐
              │     Tool.ts / tools.ts  │ ← 工具抽象 + 注册
              └────────────┬────────────┘
                           │
              ┌────────────▼────────────┐
              │  Task.ts / tasks.ts     │ ← 任务类型 + 管理
              └────────────┬────────────┘
                           │
    ┌──────────┬───────────▼──────────┬──────────┐
    │commands.ts│ cost-tracker.ts    │history.ts │ context.ts
    │(命令注册) │ (成本追踪)         │(历史管理) │(上下文构建)
    └──────────┴────────────────────┴──────────┘
```

---

## 设计优缺点总结

### 优点
1. **关注点分离**: 查询引擎、工具系统、命令系统各自独立
2. **条件编译**: `feature()` + `process.env.USER_TYPE` 实现高效的死代码消除
3. **流式架构**: AsyncGenerator 模式贯穿查询链路
4. **类型安全**: Zod schema 验证工具输入，TypeScript 泛型约束工具接口
5. **可扩展性**: 新工具/命令只需实现接口并注册

### 缺点
1. **main.tsx 过于庞大**: 4,683 行单文件，职责过多
2. **QueryEngineConfig 膨胀**: 20+ 字段的配置对象
3. **query.ts 复杂度高**: 1,729 行包含多种恢复路径和特性分支
4. **隐式依赖**: 条件导入通过 `require()` 实现循环依赖打破，增加了理解难度
5. **AppState 传播**: `getAppState/setAppState` 通过函数式更新传递，缺少类型化的 Action
