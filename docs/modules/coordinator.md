# Coordinator — 多 Agent 协调架构

## 模块概览

| 指标 | 值 |
|------|-----|
| 文件数 | 1 |
| 总代码行数 | 369 |
| 核心职责 | 定义 Coordinator-Worker 多 Agent 协调模式，控制系统提示词和工具上下文注入 |
| 关键文件 | `coordinatorMode.ts` |

### 关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `coordinatorMode.ts` | 369 | 协调模式开关、系统提示词生成、Worker 工具上下文 |

---

## 1. 架构概述

Coordinator 模式是 Claude Code 最核心的多 Agent 设计，采用 **Coordinator-Worker** 架构，由一个中央协调者（Coordinator）管理多个自治工作者（Worker）。

```
┌─────────────────────────────────────────────────┐
│                   Coordinator                    │
│  (用户对话的唯一入口)                              │
│                                                   │
│  工具:                                            │
│  ├── Agent (spawn worker)                         │
│  ├── SendMessage (continue worker)                │
│  ├── TaskStop (stop worker)                       │
│  ├── subscribe_pr_activity (PR 事件订阅)          │
│  └── 直接回答简单问题（无需委派）                   │
│                                                   │
│  职责:                                            │
│  ├── 理解用户意图                                  │
│  ├── 分解任务为子任务                               │
│  ├── 并行调度 Worker                              │
│  ├── 综合 Worker 结果                              │
│  └── 向用户汇报                                    │
└──────────┬──────────┬──────────┬─────────────────┘
           │          │          │
     ┌─────▼──┐ ┌────▼───┐ ┌───▼──────┐
     │Worker A│ │Worker B│ │Worker C   │
     │(研究)   │ │(实现)   │ │(验证)    │
     │         │ │         │ │          │
     │Bash     │ │Bash     │ │Bash      │
     │Read     │ │Read     │ │Read      │
     │Edit     │ │Edit     │ │Edit      │
     │+MCP     │ │+MCP     │ │+MCP      │
     │+Skills  │ │+Skills  │ │+Skills   │
     └─────────┘ └─────────┘ └──────────┘
```

---

## 2. 核心代码分析

### 2.1 模式激活

```typescript
// src/coordinator/coordinatorMode.ts

export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

**设计要点**:
- 双重门控：`feature('COORDINATOR_MODE')` (编译期) + 环境变量 (运行期)
- `feature()` 由 Bun bundle 的死代码消除处理，外部构建不包含此特性
- 环境变量允许同一构建在不同模式下运行

### 2.2 会话恢复时的模式匹配

```typescript
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  if (!sessionMode) return undefined

  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'

  if (currentIsCoordinator === sessionIsCoordinator) return undefined

  // 动态切换环境变量以匹配恢复的会话模式
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }

  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

**关键细节**: 恢复会话时自动匹配模式，防止 coordinator 会话被普通模式恢复（或反之），确保行为一致性。

### 2.3 Worker 工具上下文

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) return {}

  const workerTools = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
        .sort().join(', ')
    : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
        .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
        .sort().join(', ')

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

**工具分层**:
- **Simple 模式**: Worker 仅获得 Bash/Read/Edit
- **完整模式**: Worker 获得 `ASYNC_AGENT_ALLOWED_TOOLS` 中的所有工具（排除 `TEAM_CREATE/DELETE/SEND_MESSAGE/SYNTHETIC_OUTPUT` 等内部工具）
- **MCP 工具**: 自动透传给 Worker
- **Scratchpad**: 协调者和 Worker 共享的无需权限的跨 Agent 持久化空间

### 2.4 协调者系统提示词

这是整个协调架构最核心的部分（约 170 行精心设计的提示词）：

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can handle without tools

## 2. Your Tools
- **Agent** - Spawn a new worker
- **SendMessage** - Continue an existing worker
- **TaskStop** - Stop a running worker

## 3. Workers
When calling Agent, use subagent_type "worker". Workers execute tasks autonomously.

## 4. Task Workflow
| Phase | Who | Purpose |
|-------|-----|---------|
| Research | Workers (parallel) | Investigate codebase |
| Synthesis | You (coordinator) | Understand, craft specs |
| Implementation | Workers | Make changes per spec |
| Verification | Workers | Test changes work |

## 5. Writing Worker Prompts
Workers can't see your conversation. Every prompt must be self-contained.

### Always synthesize — your most important job
Never write "based on your findings" or "based on the research."
Include specific file paths, line numbers, and exactly what to change.

### Choose continue vs. spawn by context overlap
| Situation | Mechanism | Why |
|-----------|-----------|-----|
| Research explored exactly the files that need editing | Continue | Worker has files in context |
| Research was broad but implementation is narrow | Spawn fresh | Focused context is cleaner |
| Verifying code a different worker wrote | Spawn fresh | Fresh eyes, no assumptions |`
}
```

---

## 3. 任务分发与结果聚合

### 3.1 任务分发流程

```
用户: "修复 auth 模块的空指针异常"
    │
    ▼
Coordinator:
    ├── Agent({ description: "调查 auth bug",
    │          subagent_type: "worker",
    │          prompt: "调查 src/auth/ 中...不要修改文件" })
    │
    ├── Agent({ description: "研究 auth 测试",
    │          subagent_type: "worker",
    │          prompt: "查找所有测试文件...不要修改文件" })
    │
    └── "正在从两个方向调查——稍后汇报结果。"
    
    ▼ (并行执行)
    
Worker A 完成:
    <task-notification>
    <task-id>agent-a1b</task-id>
    <status>completed</status>
    <result>Found null pointer in src/auth/validate.ts:42...</result>
    </task-notification>
    
    ▼
    
Coordinator (综合分析后):
    ├── SendMessage({ to: "agent-a1b",
    │                message: "修复 src/auth/validate.ts:42 的空指针..." })
    └── "找到 Bug——正在修复中。"
```

### 3.2 结果聚合

Worker 结果以 `<task-notification>` XML 格式通过用户消息通道传递：

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

Coordinator 必须区分 `<task-notification>` 和真正的用户消息——两者都到达用户角色通道。

---

## 4. 协调者核心设计原则

### 4.1 并发策略

提示词明确指导 Coordinator 的并发行为：

> **Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible.**

- **只读任务（研究）**: 自由并行
- **写操作（实现）**: 同一文件集一次一个 Worker
- **验证**: 可在不同文件区域与实现并行

### 4.2 Prompt 合成（最重要的职责）

提示词中的核心约束：

> **Never write "based on your findings" or "based on the research." These phrases delegate understanding to the worker instead of doing it yourself.**

好的示例:
```
AgentTool({ prompt: "Fix the null pointer in src/auth/validate.ts:42. 
The user field on Session is undefined when sessions expire but the token remains cached.
Add a null check before user.id access — if null, return 401 with 'Session expired'." })
```

坏的示例:
```
AgentTool({ prompt: "Based on your findings, fix the auth bug" })  // ❌ 懒惰委派
```

### 4.3 Continue vs Spawn 决策矩阵

| 场景 | 机制 | 原因 |
|------|------|------|
| 研究探索了需要编辑的精确文件 | **Continue** | Worker 已有文件在上下文中 |
| 研究广泛但实现狭窄 | **Spawn fresh** | 避免拖入探索噪声 |
| 纠正失败或扩展最近工作 | **Continue** | Worker 有错误上下文 |
| 验证不同 Worker 的代码 | **Spawn fresh** | 新鲜视角，无假设 |
| 第一次尝试方法完全错误 | **Spawn fresh** | 错误方法上下文污染重试 |

### 4.4 验证哲学

> Verification means **proving the code works**, not confirming it exists.

- 启用功能运行测试——不仅是"测试通过"
- 运行类型检查并**调查错误**
- 独立测试——证明变更有效，不要橡皮图章

---

## 5. 与其他模块的集成

### 5.1 QueryEngine 集成

```typescript
// src/QueryEngine.ts 中的条件导入
const getCoordinatorUserContext: (...) => { [k: string]: string } =
  feature('COORDINATOR_MODE')
    ? require('./coordinator/coordinatorMode.js').getCoordinatorUserContext
    : () => ({})
```

QueryEngine 在构建用户上下文时调用 `getCoordinatorUserContext()`，将 Worker 工具信息注入系统提示。

### 5.2 tools.ts 集成

```typescript
// src/tools.ts — Coordinator 模式下追加工具
if (feature('COORDINATOR_MODE') && coordinatorModeModule?.isCoordinatorMode()) {
  simpleTools.push(AgentTool, TaskStopTool, getSendMessageTool())
}
```

### 5.3 Scratchpad 集成

Coordinator 和 Worker 共享 Scratchpad 目录，实现跨 Agent 的持久化知识共享：

```
Scratchpad directory: /path/to/scratchpad
Workers can read and write here without permission prompts.
Use this for durable cross-worker knowledge.
```

---

## 6. 设计优缺点分析

### 优点

1. **清晰的职责分离**: Coordinator 负责理解和调度，Worker 负责执行
2. **并行化设计**: 天然支持多 Worker 并行，利用异步特性
3. **合成优于委派**: 强制 Coordinator 理解结果后再下发，避免信息丢失
4. **灵活的生命周期**: Continue/Spawn 决策允许上下文复用或干净重启
5. **会话恢复安全**: `matchSessionMode()` 确保模式一致性

### 缺点

1. **单文件实现**: 所有逻辑在 369 行的 `coordinatorMode.ts` 中，系统提示词占 ~170 行，可拆分
2. **无类型化协议**: Worker 结果通过 XML 格式字符串传递，无 TypeScript 类型保护
3. **有限的状态可见性**: Coordinator 只在 Worker 完成时收到通知，无中间进度
4. **错误恢复依赖提示词**: Worker 失败处理策略完全由提示词指导，无代码级保障
5. **协调者瓶颈**: 所有 Worker 结果必须经过 Coordinator 综合，高并发场景下可能成为瓶颈

---

## 7. 关键设计模式

| 模式 | 应用 |
|------|------|
| Coordinator-Worker | 核心 Agent 协调架构 |
| Feature Flag | `feature('COORDINATOR_MODE')` 编译期 + 运行期门控 |
| 依赖注入 | `getCoordinatorUserContext()` 通过参数获取 MCP 客户端和 scratchpad 路径 |
| 零信任上下文 | Worker 无法看到 Coordinator 对话，每个 prompt 必须自包含 |
| Scratchpad | 跨 Agent 持久化共享空间，绕过权限系统 |
