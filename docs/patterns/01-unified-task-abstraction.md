# 设计模式：统一的 Task 抽象

## 1. 模式概述

**问题**：多 Agent 系统中有多种异步操作（Shell 命令、子 Agent、远程 Agent、队友协作、Dream 模式等），每种类型如果独立实现管理逻辑，会导致代码重复、状态不一致、UI 展示混乱。

**解决方案**：Claude Code 将所有异步操作统一为 `TaskState` 联合类型，共享状态机、注册机制、通知系统和 UI 展示。

**一句话总结**：一套抽象管理所有异步任务，通过多态分发实现差异化行为。

---

## 2. 源码实现分析

### 2.1 TaskType 类型系统

> 文件：`Task.ts`

Claude Code 定义了 7 种任务类型，每种对应不同的执行场景：

```typescript
export type TaskType =
  | 'local_bash'         // 本地 Shell 命令执行
  | 'local_agent'        // 本地子 Agent（进程内）
  | 'remote_agent'       // 远程 Agent（CCR 沙箱）
  | 'in_process_teammate' // 进程内队友（AsyncLocalStorage）
  | 'local_workflow'      // 本地工作流
  | 'monitor_mcp'         // MCP 监控任务
  | 'dream'              // Dream 模式（后台思考）
```

**适用场景映射**：

| TaskType | 场景 | 执行环境 | 隔离级别 |
|----------|------|---------|---------|
| `local_bash` | Shell 命令 | 本地进程 | 进程级 |
| `local_agent` | 子 Agent 任务 | 进程内（fork） | 对话级 |
| `remote_agent` | 远程沙箱任务 | CCR 远程沙箱 | 容器级 |
| `in_process_teammate` | 团队协作队友 | 进程内（AsyncLocalStorage） | 状态级 |
| `local_workflow` | 工作流编排 | 进程内 | 对话级 |
| `monitor_mcp` | MCP 服务监控 | 进程内 | 连接级 |
| `dream` | 后台思考/创意 | 后台进程 | 独立 |

### 2.2 TaskState 状态机

> 文件：`Task.ts` + `tasks/types.ts`

```typescript
// 统一的任务状态
export type TaskStatus =
  | 'pending'    // 等待执行
  | 'running'    // 执行中
  | 'completed'  // 成功完成
  | 'failed'     // 执行失败
  | 'killed'     // 被用户/系统终止

// 终态判断
export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}
```

**状态转换图**：

```
                    ┌──────────────────────────────────────┐
                    │            终态判断函数                │
                    │   isTerminalTaskStatus()              │
                    └──────────────────────────────────────┘
                                      │
     ┌──────────┐    start     ┌──────────┐    complete    ┌───────────┐
     │ pending  │ ───────────→ │ running  │ ───────────→  │ completed │
     └──────────┘              └──────────┤               └───────────┘
                                          │
                          error ┌─────────┼─────────┐ kill
                                │         │         │
                                ▼         │         ▼
                          ┌──────────┐    │   ┌──────────┐
                          │  failed  │    │   │  killed  │
                          └──────────┘    │   └──────────┘
                                          │
                                    ┌─────┴─────┐
                                    │  终态，    │
                                    │  不可转换  │
                                    └───────────┘
```

**TaskState 联合类型**（`tasks/types.ts`）：

```typescript
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState
```

每种具体状态类型都扩展自 `TaskStateBase`。

### 2.3 基础状态与注册机制

> 文件：`Task.ts`

```typescript
// 所有任务共享的基础字段
export type TaskStateBase = {
  id: string               // 任务 ID（前缀 + 8位随机）
  type: TaskType           // 任务类型
  status: TaskStatus       // 当前状态
  description: string      // 任务描述
  toolUseId?: string       // 触发的工具调用 ID
  startTime: number        // 开始时间戳
  endTime?: number         // 结束时间戳
  totalPausedMs?: number   // 暂停总时长
  outputFile: string       // 输出文件路径
  outputOffset: number     // 输出偏移量
  notified: boolean        // 是否已通知用户
}

// 工厂函数
export function createTaskStateBase(
  id: string,
  type: TaskType,
  description: string,
  toolUseId?: string,
): TaskStateBase {
  return {
    id, type,
    status: 'pending',
    description, toolUseId,
    startTime: Date.now(),
    outputFile: getTaskOutputPath(id),
    outputOffset: 0,
    notified: false,
  }
}
```

**任务 ID 生成**（`Task.ts`）：

```typescript
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',           // b + 8位随机
  local_agent: 'a',          // a + 8位随机
  remote_agent: 'r',         // r + 8位随机
  in_process_teammate: 't',  // t + 8位随机
  local_workflow: 'w',       // w + 8位随机
  monitor_mcp: 'm',          // m + 8位随机
  dream: 'd',                // d + 8位随机
}

// 36^8 ≈ 2.8万亿种组合，足以抵抗暴力符号链接攻击
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

**Task 接口**（多态分发的核心）：

```typescript
export type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}
```

注意：`kill` 是唯一的多态方法。spawn 和 render 从未被多态调用过（在 #22546 中移除），所有 6 个 kill 实现只使用 `setAppState`。

### 2.4 后台任务判断

> 文件：`tasks/types.ts`

```typescript
export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') {
    return false
  }
  // 前台任务（isBackgrounded === false）不算"后台任务"
  if ('isBackgrounded' in task && task.isBackgrounded === false) {
    return false
  }
  return true
}
```

### 2.5 各 TaskType 实现差异

| 方面 | 共同点 | 差异点 |
|------|--------|--------|
| 状态定义 | 都扩展 `TaskStateBase` | 各自附加特定字段（如 `identity`、`abortController`） |
| 状态转换 | 都遵循 pending → running → 终态 | 转换触发条件不同 |
| Kill 实现 | 都只用 `setAppState` | 资源清理逻辑不同（杀进程 vs 关 pane vs 断连接） |
| UI 展示 | 共享任务面板和进度条 | pillLabel 和图标不同 |
| 隔离方式 | — | 从无隔离（AsyncLocalStorage）到容器级（远程沙箱） |

---

## 3. 设计要点

### 3.1 开放封闭原则（OCP）

添加新 TaskType 不需要修改现有代码，只需：
1. 在 `TaskType` 联合类型中添加新成员
2. 实现 `Task` 接口的 `kill` 方法
3. 在 `TASK_ID_PREFIXES` 中添加前缀
4. 创建对应的 `TaskState` 子类型

### 3.2 状态机模式

统一的 `TaskStatus` 状态机确保所有任务的生命周期行为一致：
- 前置状态检查（不能从 completed 转为 running）
- 终态判断函数 `isTerminalTaskStatus()` 用于清理和守卫

### 3.3 观察者模式（通知）

通过 `enqueuePendingNotification` 机制，任务状态变化自动触发通知：
- 任务完成 → 通知用户
- 任务失败 → 通知用户并展示错误
- UI 层监听 AppState 变化自动更新

### 3.4 策略模式（差异化行为）

`Task` 接口的 `kill` 方法是多态的：
- `LocalShellTask.kill` → 杀 Shell 进程
- `LocalAgentTask.kill` → 中止 Agent 对话
- `InProcessTeammateTask.kill` → 关闭 tmux pane
- `RemoteAgentTask.kill` → 终止远程沙箱

---

## 4. 可落地方案

### 4.1 通用 TypeScript 实现

```typescript
// ===== 类型定义 =====
export type TaskType = 'shell' | 'agent' | 'remote' | 'teammate' | 'workflow'

export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed'

export interface TaskStateBase {
  id: string
  type: TaskType
  status: TaskStatus
  description: string
  startTime: number
  endTime?: number
  output?: string
  notified: boolean
}

export interface Task {
  name: string
  type: TaskType
  kill(taskId: string): Promise<void>
}

// ===== 状态机 =====
export function isTerminalStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed'
}

export function canTransition(from: TaskStatus, to: TaskStatus): boolean {
  if (isTerminalStatus(from)) return false
  if (from === to) return false
  const valid: Record<TaskStatus, TaskStatus[]> = {
    pending: ['running', 'killed'],
    running: ['completed', 'failed', 'killed'],
    completed: [],
    failed: [],
    killed: [],
  }
  return valid[from]?.includes(to) ?? false
}

// ===== 注册表 =====
export class TaskRegistry<T extends TaskStateBase = TaskStateBase> {
  private tasks = new Map<string, T>()
  private listeners = new Set<(task: T) => void>()

  register(task: T): void {
    this.tasks.set(task.id, task)
    this.notify(task)
  }

  update(id: string, updates: Partial<T>): T | undefined {
    const task = this.tasks.get(id)
    if (!task) return undefined
    if (!canTransition(task.status, (updates as any).status ?? task.status)) {
      throw new Error(`Invalid transition: ${task.status} → ${(updates as any).status}`)
    }
    const updated = { ...task, ...updates }
    this.tasks.set(id, updated)
    this.notify(updated)
    return updated
  }

  get(id: string): T | undefined {
    return this.tasks.get(id)
  }

  list(filter?: (task: T) => boolean): T[] {
    const all = Array.from(this.tasks.values())
    return filter ? all.filter(filter) : all
  }

  subscribe(listener: (task: T) => void): () => void {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }

  private notify(task: T): void {
    for (const listener of this.listeners) {
      listener(task)
    }
  }
}
```

### 4.2 扩展指南

添加新 TaskType 的步骤：

```typescript
// 1. 扩展类型
export type TaskType = ... | 'custom_task'

// 2. 定义状态子类型
export interface CustomTaskState extends TaskStateBase {
  type: 'custom_task'
  customField: string
}

// 3. 实现 Task 接口
export class CustomTask implements Task {
  name = 'Custom Task'
  type: TaskType = 'custom_task'

  async kill(taskId: string): Promise<void> {
    // 自定义清理逻辑
  }
}

// 4. 添加 ID 前缀
const TASK_ID_PREFIXES = { ...prefixes, custom_task: 'c' }
```

### 4.3 注意事项

1. **不要为每种类型实现独立的 UI**：共享统一面板，通过 `type` 字段差异化渲染
2. **状态转换必须经过守卫**：直接修改 status 绕过检查会导致通知遗漏
3. **kill 必须幂等**：重复调用 kill 不应报错
4. **输出写入磁盘而非内存**：长时间运行的任务输出可能很大，用文件 + offset 模式
5. **ID 前缀帮助调试**：看到 `a` 开头就知道是 agent 任务

---

## 5. 总结

Claude Code 的 Task 抽象设计体现了**统一抽象 + 多态分发**的核心理念：

1. **7 种 TaskType** 覆盖了从 Shell 命令到远程沙箱的所有异步场景
2. **统一的状态机**（pending → running → completed/failed/killed）确保生命周期行为一致
3. **TaskStateBase** 提供共享基础设施（ID、时间戳、输出文件、通知状态）
4. **Task 接口**的 kill 多态分发实现差异化资源清理
5. **ID 前缀**（b/a/r/t/w/m/d）方便调试和类型识别

这套抽象可以在任何多 Agent 系统中复用，核心价值在于**消除重复代码**和**统一生命周期管理**。
