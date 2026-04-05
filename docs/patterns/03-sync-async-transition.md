# 设计模式：同步/异步透明转换

## 1. 模式概述

**问题**：长时间运行的子任务（如复杂代码实现、大文件搜索）会阻塞主会话，用户无法继续交互，体验糟糕。

**解决方案**：运行时动态切换执行模式——任务启动时同步等待，超时后自动转为后台模式，用户可随时拉回前台。

**核心原理**：`Promise.race` 模式——同时启动前台 Promise 和后台超时 Promise，谁先完成用谁。

---

## 2. 源码实现分析

### 2.1 Promise.race 核心机制

> 文件：`tasks/LocalAgentTask/LocalAgentTask.tsx`（简化示意）

```typescript
const FOREGROUND_TIMEOUT_MS = 2 * 60 * 1000  // 2分钟

const raceResult = await Promise.race([
  nextMessagePromise.then(r => ({ type: 'message', result: r })),
  new Promise(resolve => 
    setTimeout(() => resolve({ type: 'background' }), FOREGROUND_TIMEOUT_MS)
  ),
])

if (raceResult.type === 'message') {
  // 前台收到消息，继续处理
  return raceResult.result
} else {
  // 超时，转为后台模式
  taskState.isBackgrounded = true
  setAppState(prev => ({ ...prev, tasks: { ...prev.tasks, [taskId]: taskState } }))
  // 继续执行，但不再阻塞用户
}
```

**执行流程图**：

```
任务启动
    │
    ▼
┌───────────────────────────────────────┐
│          Promise.race                  │
│  ┌─────────────┐  ┌─────────────┐     │
│  │ 前台Promise │  │ 超时Promise │     │
│  │ (等待消息)   │  │  (2分钟)    │     │
│  └──────┬──────┘  └──────┬──────┘     │
│         │                │            │
│    消息到达          超时触发          │
│         │                │            │
└─────────┼────────────────┼────────────┘
          │                │
          ▼                ▼
    继续前台处理      转为后台模式
          │                │
          │                ▼
          │         用户可继续交互
          │         任务在后台运行
          │                │
          └────────────────┼────────
                           │
                      任务完成
                           │
                           ▼
                    通知用户结果
```

### 2.2 前台注册与保持机制

```typescript
// 注册前台任务
function registerAgentForeground(
  taskId: string,
  abortController: AbortController,
): void {
  // 每次用户交互重置超时计时器
  // 用户输入、工具调用等都会触发"活跃"信号
}

// 活跃检测
function onUserActivity(): void {
  // 重置前台超时
  // 如果任务还在运行，延长前台时间
}
```

**自动转后台的触发条件**：
1. 2 分钟内无新消息（模型响应完成或工具执行中）
2. 用户长时间无交互
3. 模型主动进入长时间工具调用

### 2.3 后台任务状态管理

> 文件：`Task.ts`

```typescript
export type TaskStateBase = {
  // ... 其他字段
  notified: boolean  // 是否已通知用户结果
}

// 后台任务判断
export function isBackgroundTask(task: TaskState): task is BackgroundTaskState {
  if (task.status !== 'running' && task.status !== 'pending') {
    return false
  }
  // isBackgrounded === false 表示仍在前台
  if ('isBackgrounded' in task && task.isBackgrounded === false) {
    return false
  }
  return true
}
```

### 2.4 任务通知机制

> 文件：多处分发

后台任务完成后，通过 `<task-notification>` XML 通知用户：

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

### 2.5 取消与清理

> 文件：`tasks/stopTask.ts`

```typescript
export async function stopTask(
  taskId: string,
  setAppState: SetAppState,
): Promise<void> {
  const task = getTask(taskId)
  if (!task) return

  // 1. 触发 AbortController
  task.abortController?.abort()

  // 2. 更新状态为 killed
  setAppState(prev => ({
    ...prev,
    tasks: {
      ...prev.tasks,
      [taskId]: {
        ...prev.tasks[taskId],
        status: 'killed',
        endTime: Date.now(),
      },
    },
  }))

  // 3. 清理资源（worktree、进程、连接等）
  await task.cleanup?.()

  // 4. 发送通知
  enqueuePendingNotification({
    taskId,
    status: 'killed',
    summary: 'Task was stopped by user',
  })
}
```

---

## 3. 设计要点

### 3.1 Promise.race 模式

```
同步等待 = Promise.race([任务Promise, 超时Promise])
```

这个模式的核心是**非阻塞等待**：
- 任务在超时内完成 → 用户立即看到结果
- 任务超时 → 自动转后台，用户继续交互

### 3.2 状态双轨制

```
前台任务（isBackgrounded: false）
    │
    ├── 用户可见的进度条
    ├── 阻塞用户输入（等待完成）
    └── 可能有交互式提示
          │
          ▼ (超时触发)
后台任务（isBackgrounded: true）
    │
    ├── 用户可继续输入
    ├── 进度显示在后台任务面板
    └── 完成后通过通知告知
```

### 3.3 用户感知优先

设计原则：**用户永远不应该感觉被卡住**
- 2 分钟是感知阈值（足够短不会焦虑，足够长等待有意义）
- 转后台后任务继续，不影响结果
- 随时可以检查进度或拉回前台

### 3.4 资源安全释放

```
任务终止（用户取消/错误/完成）
    │
    ├── AbortController.abort() 中止所有异步操作
    ├── cleanup() 清理资源（进程、文件、连接）
    ├── 更新状态为终态
    └── 发送通知
```

---

## 4. 可落地方案

### 4.1 通用同步/异步转换框架

```typescript
// ===== 类型定义 =====
export type ForegroundState = 'foreground' | 'background' | 'completed'

export interface TaskRunner<T> {
  run(): Promise<T>
  onBackground?(): void
  onForeground?(): void
  cleanup?(): Promise<void>
}

// ===== 前台管理器 =====
export class ForegroundManager {
  private activeTaskId: string | null = null
  private backgroundTasks = new Map<string, AbortController>()

  startForeground(taskId: string): AbortController {
    this.activeTaskId = taskId
    const ac = new AbortController()
    this.backgroundTasks.set(taskId, ac)
    return ac
  }

  moveToBackground(taskId: string): void {
    if (this.activeTaskId === taskId) {
      this.activeTaskId = null
    }
  }

  isForeground(taskId: string): boolean {
    return this.activeTaskId === taskId
  }

  abort(taskId: string): void {
    this.backgroundTasks.get(taskId)?.abort()
    this.backgroundTasks.delete(taskId)
  }
}

// ===== 任务执行器 =====
export class AsyncTaskExecutor {
  private foregroundManager = new ForegroundManager()
  private timeoutMs: number

  constructor(timeoutMs = 120_000) {
    this.timeoutMs = timeoutMs
  }

  async execute<T>(
    taskId: string,
    runner: TaskRunner<T>,
    onProgress?: (state: ForegroundState) => void,
  ): Promise<T> {
    const ac = this.foregroundManager.startForeground(taskId)
    
    const taskPromise = (async () => {
      try {
        return await runner.run()
      } finally {
        runner.cleanup?.()
      }
    })()

    const timeoutPromise = new Promise<'timeout'>(resolve =>
      setTimeout(() => resolve('timeout'), this.timeoutMs)
    )

    // 前台等待
    const raceResult = await Promise.race([
      taskPromise.then(r => ({ type: 'result' as const, value: r })),
      timeoutPromise.then(() => ({ type: 'timeout' as const })),
    ])

    if (raceResult.type === 'result') {
      this.foregroundManager.moveToBackground(taskId)
      onProgress?.('completed')
      return raceResult.value
    }

    // 超时，转后台
    this.foregroundManager.moveToBackground(taskId)
    runner.onBackground?.()
    onProgress?.('background')

    // 继续等待，但不阻塞
    return taskPromise.then(result => {
      onProgress?.('completed')
      return result
    })
  }

  abort(taskId: string): void {
    this.foregroundManager.abort(taskId)
  }
}
```

### 4.2 集成到 Agent 系统

```typescript
// 在 Agent 工具中使用
export class AgentTool {
  private executor = new AsyncTaskExecutor()

  async call(input: AgentInput, context: ToolContext): Promise<ToolResult> {
    const taskId = generateTaskId('local_agent')

    // 异步执行，不阻塞用户
    const resultPromise = this.executor.execute(
      taskId,
      {
        run: () => this.runAgent(input, context),
        onBackground: () => {
          // 更新 UI 显示后台状态
          context.updateUI({ taskId, state: 'background' })
        },
        cleanup: async () => {
          // 清理 worktree、断开连接等
          await this.cleanup(taskId)
        },
      },
      state => {
        // 进度回调
        context.notify({ taskId, state })
      }
    )

    // 返回 Promise，让调用方决定如何处理
    return resultPromise
  }
}
```

### 4.3 高级场景

**多层嵌套转换**：
```typescript
// 父任务转后台后，子任务也应该转后台
if (parentState === 'background') {
  childTask.startAsBackground()
}
```

**用户主动拉回前台**：
```typescript
function bringToForeground(taskId: string): void {
  const task = getTask(taskId)
  if (task && task.status === 'running') {
    task.isBackgrounded = false
    // 重置超时计时器
    resetForegroundTimeout(taskId)
  }
}
```

---

## 5. 总结

Claude Code 的同步/异步透明转换设计体现了**用户感知优先**的理念：

1. **Promise.race 模式**：2 分钟超时是用户感知的临界点
2. **自动转后台**：用户永远不会感觉被卡住
3. **状态双轨制**：前台任务有进度条，后台任务有通知
4. **AbortController**：安全中止所有异步操作
5. **资源清理**：转后台或取消时确保资源释放

这套设计让多 Agent 系统既能并行执行复杂任务，又不影响用户的即时交互体验。
