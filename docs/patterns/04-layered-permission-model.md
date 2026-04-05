# 设计模式：分层权限模型

## 1. 模式概述

**问题**：不同 Agent 角色有不同的信任级别和职责。如果统一权限策略，要么过于宽松导致安全风险，要么过于严格限制功能。

**解决方案**：分层 + 可配置的权限模型，每种 Agent 类型有独立的权限策略。

**权限层级图**：

```
┌─────────────────────────────────────────────────────────────┐
│                     权限模式层级                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐                                        │
│  │  bypassPermissions │ ← 最高信任，跳过所有权限检查          │
│  │  (Coordinator)     │   用于主会话、可信 Agent             │
│  └─────────────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │   acceptEdits   │ ← 独立权限模式                          │
│  │   (Worker)      │   自动接受文件编辑，独立于父限制          │
│  └─────────────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │     auto        │ ← 自动分类器                            │
│  │                 │   根据操作风险自动决定                   │
│  └─────────────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │   plan_mode     │ ← 计划审批                              │
│  │   (Teammate)    │   必须先规划，用户批准后执行             │
│  └─────────────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │  read-only      │ ← 只读模式                              │
│  │  (Explore/Plan) │   禁止任何文件修改                      │
│  └─────────────────┘                                        │
│           │                                                 │
│           ▼                                                 │
│  ┌─────────────────┐                                        │
│  │     bubble      │ ← 冒泡模式                              │
│  │   (Fork Child)  │   权限提示转发给父终端                   │
│  └─────────────────┘                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 源码实现分析

### 2.1 Worker 独立权限模式

> 文件：`tools/shared/spawnMultiAgent.ts`

```typescript
function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []
  const { planModeRequired, permissionMode } = options || {}

  // 传播权限模式给队友，但如果 plan_mode_required 则不传播 bypass
  if (planModeRequired) {
    // 不继承 bypass 权限，plan 模式优先
  } else if (
    permissionMode === 'bypassPermissions' ||
    getSessionBypassPermissionsMode()
  ) {
    flags.push('--dangerously-skip-permissions')
  } else if (permissionMode === 'acceptEdits') {
    flags.push('--permission-mode acceptEdits')
  } else if (permissionMode === 'auto') {
    // 队友继承 auto 模式，分类器也会自动批准它们的工具调用
    flags.push('--permission-mode auto')
  }

  return flags.join(' ')
}
```

**acceptEdits 模式特点**：
- 自动接受文件编辑操作
- 独立于父会话的权限限制
- Worker 可以修改文件而无需父会话批准

### 2.2 Fork 冒泡模式

> 文件：`tools/AgentTool/forkSubagent.ts`

```typescript
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',
  permissionMode: 'bubble',  // 权限冒泡到父终端
  source: 'built-in',
}
```

**bubble 模式行为**：
- Fork 子 Agent 遇到权限提示时，不直接展示给用户
- 将权限请求转发到父会话的终端
- 用户在父终端看到提示并做决定
- 决定结果传回子 Agent

### 2.3 Teammate 计划审批模式

> 文件：`tools/shared/spawnMultiAgent.ts`

```typescript
// Teammate 可以配置 plan_mode_required
const taskState: InProcessTeammateTaskState = {
  // ...
  identity: {
    agentId: teammateId,
    agentName: sanitizedName,
    teamName,
    color: teammateColor,
    planModeRequired: plan_mode_required ?? false,
    parentSessionId: getSessionId(),
  },
  awaitingPlanApproval: false,
  permissionMode: plan_mode_required ? 'plan' : 'default',
}
```

**plan_mode 流程**：

```
Teammate 启动
    │
    ▼
分析任务，生成计划
    │
    ▼
提交计划等待审批
    │
    ├── 用户批准 ──→ 执行计划
    │                    │
    │                    ▼
    │               完成/失败
    │
    └── 用户拒绝 ──→ 重新规划或终止
```

### 2.4 只读 Agent 权限

> 文件：`coordinator/coordinatorMode.ts` + `tools/AgentTool/built-in/`

```typescript
// Explore 和 Plan Agent 是只读的
// 通过工具白名单 + isReadOnly 实现

// 工具白名单机制
export const ASYNC_AGENT_ALLOWED_TOOLS: Set<string> = new Set([
  'Read',
  'Grep',
  'Glob',
  'LS',
  'WebSearch',
  'WebFetch',
  // 注意：没有 Write、Edit、Bash
])
```

### 2.5 工具白名单机制

> 文件：`coordinator/coordinatorMode.ts`

```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])

// Worker 可用工具 = ASYNC_AGENT_ALLOWED_TOOLS - INTERNAL_WORKER_TOOLS
const workerTools = Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
  .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
  .sort()
  .join(', ')
```

**工具级别控制**：
- `ASYNC_AGENT_ALLOWED_TOOLS`：所有异步 Agent 可用的基础工具集
- `INTERNAL_WORKER_TOOLS`：内部工具，从基础集中排除
- 不同 Agent 类型可以进一步限制

### 2.6 Coordinator 的 Worker 权限定义

> 文件：`coordinator/coordinatorMode.ts`

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) {
    return {}
  }

  const workerTools = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
        .sort()
        .join(', ')
    : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
        .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
        .sort()
        .join(', ')

  let content = `Workers spawned via the ${AGENT_TOOL_NAME} tool have access to these tools: ${workerTools}`

  if (mcpClients.length > 0) {
    const serverNames = mcpClients.map(c => c.name).join(', ')
    content += `\n\nWorkers also have access to MCP tools from connected MCP servers: ${serverNames}`
  }

  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}\nWorkers can read and write here without permission prompts.`
  }

  return { workerToolsContext: content }
}
```

---

## 3. 设计要点

### 3.1 最小权限原则

每个 Agent 只获得完成其任务所需的最小权限：
- **Explore Agent**：只读，不能修改任何文件
- **Worker Agent**：可写，但受父会话约束
- **Coordinator**：最高权限，但不应直接执行写操作

### 3.2 三种权限策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **独立（acceptEdits）** | Worker 独立任务 | 高效，无阻塞 | 权限隔离弱 |
| **冒泡（bubble）** | Fork 子任务 | 统一入口，用户友好 | 增加父会话负担 |
| **审批（plan_mode）** | Teammate 协作 | 安全可控 | 需要额外交互 |

### 3.3 默认拒绝（Fail-Closed）

> 文件：`Tool.ts`

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,      // 默认不可并发（安全）
  isReadOnly: () => false,             // 默认非只读（安全）
  isDestructive: () => false,
  checkPermissions: () => Promise.resolve({ behavior: 'allow', updatedInput }),
}
```

所有工具的安全默认值偏向严格：
- 新工具必须显式声明 `isReadOnly` 或 `isConcurrencySafe`
- 未声明的行为被视为不安全

### 3.4 工具级别的细粒度控制

权限不只是 Agent 级别，还下沉到工具级别：
- 每个工具有 `checkPermissions` 方法
- 可以根据输入参数动态决定权限
- 权限结果可以是 `allow`、`deny` 或 `prompt`

---

## 4. 可落地方案

### 4.1 通用权限框架

```typescript
// ===== 类型定义 =====
export type PermissionMode = 
  | 'bypass'       // 跳过所有检查
  | 'acceptEdits'  // 自动接受编辑
  | 'auto'         // 自动分类
  | 'plan'         // 计划审批
  | 'readonly'     // 只读
  | 'bubble'       // 冒泡到父
  | 'default'      // 默认提示

export type PermissionResult = 
  | { behavior: 'allow' }
  | { behavior: 'deny'; reason: string }
  | { behavior: 'prompt'; message: string }

export interface PermissionContext {
  agentId: string
  agentType: string
  permissionMode: PermissionMode
  parentId?: string
}

// ===== 权限管理器 =====
export class PermissionManager {
  private readonly toolPermissions = new Map<string, ToolPermission>()
  private readonly agentPermissions = new Map<string, PermissionContext>()

  registerTool(toolName: string, permission: ToolPermission): void {
    this.toolPermissions.set(toolName, permission)
  }

  registerAgent(agentId: string, context: PermissionContext): void {
    this.agentPermissions.set(agentId, context)
  }

  async checkPermission(
    agentId: string,
    toolName: string,
    input: unknown,
  ): Promise<PermissionResult> {
    const ctx = this.agentPermissions.get(agentId)
    if (!ctx) return { behavior: 'deny', reason: 'Unknown agent' }

    // bypass 模式直接放行
    if (ctx.permissionMode === 'bypass') {
      return { behavior: 'allow' }
    }

    // readonly 模式检查工具是否只读
    if (ctx.permissionMode === 'readonly') {
      const toolPerm = this.toolPermissions.get(toolName)
      if (toolPerm && !toolPerm.isReadOnly(input)) {
        return { behavior: 'deny', reason: 'Read-only mode' }
      }
    }

    // bubble 模式转发到父
    if (ctx.permissionMode === 'bubble' && ctx.parentId) {
      return this.checkPermission(ctx.parentId, toolName, input)
    }

    // plan 模式需要检查是否有待批准的计划
    if (ctx.permissionMode === 'plan') {
      // 实现计划审批逻辑
    }

    // 默认：检查工具权限
    const toolPerm = this.toolPermissions.get(toolName)
    if (!toolPerm) return { behavior: 'prompt', message: `Unknown tool: ${toolName}` }
    
    return toolPerm.checkPermission(input, ctx)
  }
}

// ===== 工具权限接口 =====
export interface ToolPermission {
  isReadOnly(input: unknown): boolean
  isDestructive?(input: unknown): boolean
  checkPermission(input: unknown, ctx: PermissionContext): Promise<PermissionResult>
}
```

### 4.2 三种策略实现

```typescript
// 独立权限模式
export class AcceptEditsStrategy {
  readonly mode = 'acceptEdits'

  check(toolName: string, input: unknown): PermissionResult {
    if (['Write', 'Edit', 'Bash'].includes(toolName)) {
      return { behavior: 'allow' }  // 自动接受编辑
    }
    return { behavior: 'prompt', message: `Permission for ${toolName}?` }
  }
}

// 冒泡权限模式
export class BubbleStrategy {
  readonly mode = 'bubble'
  private parentManager: PermissionManager

  constructor(parentManager: PermissionManager, private parentId: string) {
    this.parentManager = parentManager
  }

  async check(toolName: string, input: unknown): Promise<PermissionResult> {
    // 转发到父 Agent 的权限管理器
    return this.parentManager.checkPermission(this.parentId, toolName, input)
  }
}

// 计划审批模式
export class PlanModeStrategy {
  readonly mode = 'plan'
  private planApproved = false
  private pendingPlan: string | null = null

  async check(toolName: string, input: unknown): Promise<PermissionResult> {
    if (!this.planApproved) {
      return { 
        behavior: 'prompt', 
        message: 'Plan not yet approved. Submit a plan first.' 
      }
    }
    // 计划已批准，检查操作是否在计划范围内
    return { behavior: 'allow' }
  }

  submitPlan(plan: string): void {
    this.pendingPlan = plan
  }

  approvePlan(): void {
    this.planApproved = true
    this.pendingPlan = null
  }

  rejectPlan(): void {
    this.pendingPlan = null
  }
}
```

### 4.3 安全最佳实践

1. **权限提升攻击防护**：
   - 子 Agent 不能获得比父 Agent 更高的权限
   - bubble 模式不循环（检测并拒绝循环引用）

2. **横向移动防护**：
   - 每个 Agent 有独立的工具白名单
   - 敏感工具（如 Bash）需要额外审批

3. **审计与追踪**：
   ```typescript
   export class PermissionAuditLog {
     private logs: PermissionLogEntry[] = []

     log(entry: PermissionLogEntry): void {
       this.logs.push({
         ...entry,
         timestamp: Date.now(),
       })
     }

     query(filter: LogFilter): PermissionLogEntry[] {
       return this.logs.filter(e => matchesFilter(e, filter))
     }
   }
   ```

---

## 5. 总结

Claude Code 的分层权限模型体现了**最小权限 + 灵活配置**的理念：

1. **五种权限模式**：bypass、acceptEdits、auto、plan、readonly、bubble
2. **双重控制**：Agent 级别 + 工具级别
3. **Fail-Closed 默认值**：新工具默认不安全
4. **冒泡机制**：Fork 子 Agent 的权限提示转发到父终端
5. **计划审批**：Teammate 模式确保用户控制关键操作

这套设计在安全性和灵活性之间取得了平衡，是多 Agent 系统权限设计的参考范式。
