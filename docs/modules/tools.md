# Tools 模块深度分析

## 模块概览

| 指标 | 值 |
|------|-----|
| 文件数 | ~184 |
| 核心职责 | 定义所有 LLM 可调用的工具（Tool），提供统一的注册、权限控制、执行和渲染机制 |
| 关键文件 | `src/Tool.ts`, `src/tools.ts`, `src/tools/BashTool/`, `src/tools/FileEditTool/`, `src/tools/FileReadTool/`, `src/tools/MCPTool/`, `src/services/tools/` |

### 关键文件列表

| 文件路径 | 作用 |
|----------|------|
| `src/Tool.ts` | Tool 类型定义、`buildTool` 工厂、`ToolUseContext` 上下文 |
| `src/tools.ts` | 所有工具的注册入口 `getAllBaseTools()` |
| `src/tools/BashTool/` | Bash 命令执行工具（最复杂的工具） |
| `src/tools/FileReadTool/` | 文件读取工具 |
| `src/tools/FileEditTool/` | 文件编辑工具 |
| `src/tools/FileWriteTool/` | 文件写入工具 |
| `src/tools/GrepTool/` | 正则搜索工具（基于 ripgrep） |
| `src/tools/GlobTool/` | 文件名匹配工具 |
| `src/tools/MCPTool/` | MCP 协议工具桥接 |
| `src/tools/AgentTool/` | 子代理（subagent）工具 |
| `src/services/tools/toolOrchestration.ts` | 工具编排与并发调度 |
| `src/services/tools/toolExecution.ts` | 单个工具的执行流程 |
| `src/services/tools/StreamingToolExecutor.ts` | 流式工具执行器 |

---

## 1. 完整工具清单

所有工具均通过 `src/tools.ts` 中的 `getAllBaseTools()` 注册：

### 1.1 核心/通用工具（始终可用）

| 工具名 | 目录 | 功能 | 只读 |
|--------|------|------|------|
| `Bash` | `BashTool/` | 执行 shell 命令 | 否 |
| `Read` | `FileReadTool/` | 读取文件内容（支持文本、图片、PDF） | 是 |
| `Edit` | `FileEditTool/` | 精确文本替换编辑文件 | 否 |
| `Write` | `FileWriteTool/` | 创建或覆盖文件 | 否 |
| `Glob` | `GlobTool/` | 按通配符模式匹配文件名 | 是 |
| `Grep` | `GrepTool/` | 正则搜索文件内容（基于 ripgrep） | 是 |
| `NotebookEdit` | `NotebookEditTool/` | 编辑 Jupyter notebook 单元格 | 否 |
| `WebFetch` | `WebFetchTool/` | 抓取网页内容 | 是 |
| `WebSearch` | `WebSearchTool/` | 网络搜索 | 是 |
| `TodoWrite` | `TodoWriteTool/` | 管理待办事项列表 | 否 |
| `Agent` | `AgentTool/` | 启动子代理（subagent） | 否 |
| `AskUserQuestion` | `AskUserQuestionTool/` | 向用户提问 | 是 |
| `Skill` | `SkillTool/` | 调用已注册的技能 | 否 |
| `SendUserMessage` | `BriefTool/` | 发送消息给用户 | 是 |
| `SendMessage` | `SendMessageTool/` | 向代理/频道发送消息 | 否 |
| `ToolSearch` | `ToolSearchTool/` | 搜索和加载延迟加载的工具 | 是 |
| `ListMcpResourcesTool` | `ListMcpResourcesTool/` | 列出 MCP 资源 | 是 |
| `ReadMcpResourceTool` | `ReadMcpResourceTool/` | 读取 MCP 资源 | 是 |
| `EnterPlanMode` | `EnterPlanModeTool/` | 进入计划模式 | 否 |
| `ExitPlanMode` | `ExitPlanModeTool/` | 退出计划模式 | 否 |
| `TaskOutputTool` | `TaskOutputTool/` | 获取任务输出 | 是 |
| `TaskStop` | `TaskStopTool/` | 停止任务 | 否 |

### 1.2 条件启用工具（Feature Flag / 环境变量控制）

| 工具名 | 条件 | 功能 |
|--------|------|------|
| `Config` | `USER_TYPE === 'ant'` | 读取/修改配置 |
| `TungstenTool` | `USER_TYPE === 'ant'` | Tungsten 内部工具 |
| `REPL` | `USER_TYPE === 'ant'` | REPL 交互式工具 |
| `SuggestBackgroundPR` | `USER_TYPE === 'ant'` | 建议后台 PR |
| `PowerShell` | `isPowerShellToolEnabled()` | Windows PowerShell 执行 |
| `EnterWorktree` / `ExitWorktree` | `isWorktreeModeEnabled()` | Git worktree 管理 |
| `LSP` | `ENABLE_LSP_TOOL` | 语言服务器协议工具 |
| `CronCreate/CronDelete/CronList` | `AGENT_TRIGGERS` | 定时任务管理 |
| `RemoteTrigger` | `AGENT_TRIGGERS_REMOTE` | 远程触发工具 |
| `MonitorTool` | `MONITOR_TOOL` | 监控工具 |
| `WebBrowser` | `WEB_BROWSER_TOOL` | 浏览器自动化 |
| `Workflow` | `WORKFLOW_SCRIPTS` | 工作流执行 |
| `Sleep` | `PROACTIVE` 或 `KAIROS` | 延迟工具 |
| `SendUserFile` | `KAIROS` | 发送文件给用户 |
| `PushNotification` | `KAIROS` | 推送通知 |
| `SubscribePR` | `KAIROS_GITHUB_WEBHOOKS` | PR webhook 订阅 |
| `SnipTool` | `HISTORY_SNIP` | 历史裁剪 |
| `ListPeers` | `UDS_INBOX` | 对等节点列表 |
| `TeamCreate/TeamDelete` | `isAgentSwarmsEnabled()` | 团队管理 |
| `TaskCreate/TaskGet/TaskUpdate/TaskList` | `isTodoV2Enabled()` | V2 任务系统 |
| `VerifyPlanExecution` | `CLAUDE_CODE_VERIFY_PLAN` | 计划执行验证 |

### 1.3 MCP 动态工具

MCP 工具不在 `getAllBaseTools()` 中静态注册，而是在 `src/services/mcp/client.ts` 中动态创建：

```typescript
// src/services/mcp/client.ts
// 当 MCP 服务器连接后，动态为每个 MCP 工具创建 Tool 实例
// 使用 MCPTool 作为基础模板，覆写 name、description、inputSchema 等
```

MCP 工具的命名规则为 `mcp__<serverName>__<toolName>`。

---

## 2. 工具注册机制

### 2.1 `buildTool` 工厂函数

所有工具通过 `buildTool()` 工厂函数创建，定义在 `src/Tool.ts` 中：

```typescript
// src/Tool.ts (line 757-788)
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (input, _ctx) =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

**设计要点：**
- `ToolDef` 类型允许省略默认方法，`buildTool` 自动填充
- 返回的 `BuiltTool<D>` 保证所有方法都存在（类型安全）
- 默认值遵循 **fail-closed** 原则：`isConcurrencySafe` 默认 `false`，`isReadOnly` 默认 `false`

### 2.2 工具注册流程

```
getAllBaseTools() [src/tools.ts]
    ↓ 返回 Tools 数组（所有静态工具）
getTools() [查询时调用]
    ↓ 合并 base tools + MCP tools
    ↓ 过滤 isEnabled() + deny rules
最终传递给 API 请求
```

关键代码 `src/tools.ts`：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool, BashTool, FileReadTool, FileEditTool, FileWriteTool,
    GlobTool, GrepTool, // ... 等
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ...(isToolSearchEnabledOptimistic() ? [ToolSearchTool] : []),
    // MCP 工具在运行时动态添加
  ]
}
```

### 2.3 Tool 类型结构

每个工具实现以下核心接口（`src/Tool.ts`）：

```typescript
type Tool<Input, Output, P> = {
  name: string
  aliases?: string[]            // 别名（向后兼容）
  inputSchema: Input            // Zod schema
  outputSchema?: z.ZodType
  maxResultSizeChars: number    // 结果持久化阈值

  // 核心方法
  call(args, context, canUseTool, parentMessage, onProgress?): Promise<ToolResult<Output>>
  description(input, options): Promise<string>
  prompt(options): Promise<string>

  // 权限控制
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>

  // 行为标记
  isEnabled(): boolean
  isReadOnly(input): boolean
  isConcurrencySafe(input): boolean
  isDestructive?(input): boolean
  isOpenWorld?(input): boolean

  // UI 渲染
  renderToolUseMessage(input, options): ReactNode
  renderToolResultMessage?(content, ...): ReactNode
  mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam

  // 其他
  interruptBehavior?(): 'cancel' | 'block'
  isSearchOrReadCommand?(input): { isSearch, isRead, isList }
  shouldDefer?: boolean         // 延迟加载
  alwaysLoad?: boolean          // 始终加载
  mcpInfo?: { serverName, toolName }
}
```

---

## 3. 权限控制模型

### 3.1 多层权限检查

工具权限控制是一个多层次防御系统：

```
用户消息 → API 返回 tool_use
    ↓
1. validateInput()       — 输入验证（文件存在性、路径合法性等）
    ↓ 失败 → 返回错误
2. checkPermissions()    — 工具级权限检查
    ↓ 返回 { behavior, updatedInput }
3. canUseTool()          — 通用权限系统（用户交互/自动批准/拒绝）
    ↓ 显示权限对话框或自动决定
4. Pre-tool hooks        — 钩子拦截
    ↓
5. 执行 call()
```

### 3.2 权限结果类型

```typescript
// src/utils/permissions/PermissionResult.ts
type PermissionResult =
  | { behavior: 'allow'; updatedInput?: any }
  | { behavior: 'deny'; message: string }
  | { behavior: 'passthrough'; message: string }
```

- **allow**: 直接允许，可选修改输入
- **deny**: 拒绝执行，返回错误消息
- **passthrough**: 交给上层权限系统决定

### 3.3 工具权限标记

三个关键标记决定了工具的运行方式：

| 标记 | 默认值 | 影响 |
|------|--------|------|
| `isReadOnly(input)` | `false` | 为 `true` 时可与其他只读工具并发执行 |
| `isDestructive(input)` | `false` | 标记破坏性操作（删除、覆盖、发送） |
| `isConcurrencySafe(input)` | `false` | 为 `true` 时可与其他安全工具并行 |

### 3.4 Bash 工具的特殊权限模型

Bash 工具有最复杂的权限系统，包含：

**文件路径**：`src/tools/BashTool/bashPermissions.ts`

```typescript
// 权限检查链：
// 1. 解析命令 → 提取基础命令和参数
// 2. 匹配 alwaysAllow/alwaysDeny 规则
// 3. Bash classifier（ML 分类器）自动判断安全性
// 4. 用户交互确认
```

**安全检查**（`src/tools/BashTool/bashSecurity.ts`）：

- 命令注入检测（命令替换 `$()`、进程替换 `<()`）
- Zsh 特殊命令检测（`zmodload`、`emulate`、`sysopen` 等）
- IFS 注入检测
- 混淆检测（Unicode 空白字符、控制字符）
- 路径约束检查（`checkPathConstraints`）
- Sandbox 模式（`SandboxManager`）

---

## 4. 工具执行流程

### 4.1 执行管道

```
API streaming → tool_use block 接收
    ↓
StreamingToolExecutor.addTool()     [src/services/tools/StreamingToolExecutor.ts]
    ↓ 判断 isConcurrencySafe
    ↓
partitionToolCalls()                [src/services/tools/toolOrchestration.ts]
    ↓ 分组为安全批次 和排他批次
    ↓
runToolUse()                        [src/services/tools/toolExecution.ts]
    ↓ validateInput → checkPermissions → canUseTool → hooks → call()
    ↓
ToolResult → mapToolResultToToolResultBlockParam()
    ↓
发送 tool_result 给 API
```

### 4.2 并发控制

`StreamingToolExecutor` 实现了精细的并发控制：

```typescript
// src/services/tools/StreamingToolExecutor.ts
export class StreamingToolExecutor {
  addTool(block, assistantMessage): void {
    // 根据工具的 isConcurrencySafe 属性分组
    // 安全工具可以并行执行
    // 非安全工具必须串行执行
  }
}
```

编排逻辑（`src/services/tools/toolOrchestration.ts`）：

```typescript
export async function* runTools(toolUseMessages, ...) {
  for (const { isConcurrencySafe, blocks } of partitionToolCalls(...)) {
    if (isConcurrencySafe) {
      // 并发执行只读工具批次
      yield* runToolsConcurrently(blocks, ...)
    } else {
      // 串行执行写入工具
      yield* runToolsSerially(blocks, ...)
    }
  }
}
```

### 4.3 上下文传递

`ToolUseContext` 是贯穿工具执行的核心上下文对象：

```typescript
type ToolUseContext = {
  options: {
    commands: Command[]
    tools: Tools
    mcpClients: MCPServerConnection[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    // ...
  }
  abortController: AbortController
  readFileState: FileStateCache
  messages: Message[]
  // 状态管理
  getAppState(): AppState
  setAppState(f): void
  // UI 回调
  setToolJSX?: SetToolJSXFn
  addNotification?: (notif) => void
  // ...
}
```

---

## 5. 工具与 MCP 的关系

### 5.1 MCP 工具桥接

MCP（Model Context Protocol）工具通过 `MCPTool` 模板动态创建：

**文件**：`src/tools/MCPTool/MCPTool.ts`

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',
  maxResultSizeChars: 100_000,
  inputSchema: z.object({}).passthrough(), // 接受任意输入
  // call、description、prompt 等在 mcpClient.ts 中动态覆写
})
```

### 5.2 MCP 工具生命周期

```
1. MCP 服务器配置加载 [src/services/mcp/config.ts]
    ↓
2. 客户端连接 [src/services/mcp/client.ts]
    ↓
3. listTools() → 获取工具列表
    ↓
4. 为每个 MCP 工具创建 Tool 实例（基于 MCPTool 模板）
    ↓ 覆写 name → "mcp__serverName__toolName"
    ↓ 覆写 inputSchema → 从 MCP schema 转换
    ↓ 覆写 call() → 通过 MCP client.callTool() 执行
5. 注册到工具列表
    ↓
6. 用户通过 /mcp 命令管理服务器
```

### 5.3 MCP 传输层

支持的传输类型（`src/services/mcp/types.ts`）：

- `stdio` — 标准输入输出
- `sse` — Server-Sent Events
- `sse-ide` — IDE 特化的 SSE
- `http` — HTTP 流式传输
- `ws` — WebSocket
- `sdk` — SDK 直接调用

---

## 6. 工具结果渲染

### 6.1 渲染方法

每个工具定义多个 UI 渲染方法：

| 方法 | 作用 |
|------|------|
| `renderToolUseMessage(input, options)` | 渲染工具调用消息（执行中） |
| `renderToolResultMessage(content, ...)` | 渲染工具结果消息 |
| `renderToolUseErrorMessage(result, ...)` | 渲染错误消息 |
| `renderToolUseRejectedMessage(input, ...)` | 渲染拒绝消息 |
| `renderToolUseProgressMessage(...)` | 渲染进度消息 |
| `renderToolUseTag(input)` | 渲染标签（超时、模型等元数据） |
| `renderGroupedToolUse(...)` | 渲染并行的工具组 |
| `getToolUseSummary(input)` | 紧凑视图的摘要 |
| `getActivityDescription(input)` | Spinner 显示的活动描述 |

### 6.2 工具结果持久化

当工具输出超过 `maxResultSizeChars` 阈值时，结果会被持久化到磁盘：

```typescript
type Tool = {
  maxResultSizeChars: number  // 结果持久化阈值（字符数）
  // 超过此大小 → 保存到文件 → 返回文件路径预览
  // 设为 Infinity → 永不持久化（如 Read 工具避免循环）
}
```

### 6.3 结果到 API 的映射

```typescript
mapToolResultToToolResultBlockParam(content, toolUseID): ToolResultBlockParam
```

将工具输出转换为 Anthropic API 的 `tool_result` 块。

---

## 7. 模块架构图

```
                          ┌─────────────────────┐
                          │     Anthropic API    │
                          │   (tool_use block)   │
                          └──────────┬──────────┘
                                     │
                          ┌──────────▼──────────┐
                          │  StreamingToolExecutor│
                          │  (并发控制 + 流式执行) │
                          └──────────┬──────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                 │
          ┌─────────▼──────┐ ┌──────▼───────┐ ┌──────▼────────┐
          │  partitionSafe  │ │  runToolUse  │ │  partitionSafe│
          │  (并发只读批次)  │ │  (单个执行)  │ │ (串行写入批次) │
          └────────┬───────┘ └──────┬───────┘ └──────┬────────┘
                   │                │                 │
                   └────────────────┼─────────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │       Tool 执行管道            │
                    │  validateInput                 │
                    │  → checkPermissions            │
                    │  → canUseTool (用户确认)        │
                    │  → Pre-tool hooks              │
                    │  → tool.call()                 │
                    │  → Post-tool hooks             │
                    └───────────────┬───────────────┘
                                    │
          ┌────────────┬───────────┼──────────┬─────────────┐
          │            │           │          │             │
    ┌─────▼────┐ ┌─────▼────┐ ┌───▼────┐ ┌──▼─────┐ ┌─────▼────┐
    │BashTool  │ │Read/Edit │ │GrepTool│ │MCPTool │ │AgentTool │
    │(安全检查) │ │(文件操作) │ │(搜索)  │ │(外部)  │ │(子代理)  │
    └──────────┘ └──────────┘ └────────┘ └────────┘ └──────────┘
```

---

## 8. 设计优缺点分析

### 优点

1. **统一的工具接口**：`buildTool` + `TOOL_DEFAULTS` 确保所有工具具有一致的 API，默认值遵循 fail-closed 原则
2. **精细的并发控制**：`isConcurrencySafe` / `isReadOnly` 标记 + `StreamingToolExecutor` 实现了高效的并行执行
3. **多层权限防御**：`validateInput` → `checkPermissions` → `canUseTool` → hooks 形成纵深防御
4. **Bash 安全分析极尽完善**：Tree-sitter AST 解析、Zsh 特殊命令检测、混淆检测、沙箱模式
5. **MCP 动态桥接**：通过模板工具 + 动态覆写的方式优雅地桥接 MCP 协议
6. **Feature Flag 驱动**：工具的可用性通过 Bun 的 DCE `feature()` 控制，构建时可裁剪不需要的工具
7. **工具延迟加载**：`ToolSearch` 机制允许工具 schema 延迟发送，减少首次 prompt token 消耗

### 缺点

1. **`Tool.ts` 过于庞大**（~800 行）：类型定义、上下文、工厂函数全在一个文件中，职责混杂
2. **BashTool 极其复杂**：权限检查逻辑散布在 `bashPermissions.ts`、`bashSecurity.ts`、`bashCommandHelpers.ts` 等多个文件中，总代码量巨大
3. **Feature Flag 耦合**：工具注册高度依赖 `feature()` 和 `process.env`，条件注册逻辑分散
4. **循环依赖问题**：`tools.ts` 中多处使用 `require()` 懒加载来打破循环依赖
5. **`ToolUseContext` 过度膨胀**：包含 40+ 字段，承载了太多不同层次的关注点
6. **MCP 工具覆写模式不够类型安全**：运行时动态覆写 MCPTool 的方法，容易引入不一致
7. **权限规则匹配复杂**：`alwaysAllow` / `alwaysDeny` / `alwaysAsk` 规则来源多样（用户设置、hooks、分类器），调试困难
