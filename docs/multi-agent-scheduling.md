# Claude Code 多 Agent 并行调度设计深度分析

## 1. 架构总览

### 1.1 多 Agent 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户输入 (User Input)                         │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     QueryEngine (查询引擎)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │  submitMsg() │→│  query()     │→│  SystemPrompt 组装       │    │
│  └──────────────┘  └──────────────┘  │  + Coordinator 模式注入 │    │
│                                      └────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
              ┌────────────┼────────────────┐
              │            │                │
              ▼            ▼                ▼
┌──────────────┐ ┌──────────────┐  ┌──────────────────┐
│ Coordinator  │ │  Agent Tool  │  │  Fork Subagent   │
│    Mode      │ │  (标准子Agent)│  │  (隐式分叉路径)   │
│ ┌──────────┐ │ └──────┬───────┘  └────────┬─────────┘
│ │ 调度策略 │ │        │                    │
│ │ Worker   │ │        ▼                    ▼
│ │ SendMessage│ │  ┌──────────────────────────────────┐
│ │ TaskStop  │ │  │        runAgent() 核心             │
│ └──────────┘ │  │  ┌─────────────┐ ┌──────────────┐ │
└──────────────┘  │  │Tool Pool组装 │ │ Permission   │ │
                  │  │(assembleTool)│ │ Mode Override│ │
                  │  └─────────────┘ └──────────────┘ │
                  │  ┌─────────────┐ ┌──────────────┐ │
                  │  │MCP Server   │ │ Agent Memory │ │
                  │  │ 初始化       │ │ 加载         │ │
                  │  └─────────────┘ └──────────────┘ │
                  │  ┌─────────────────────────────┐   │
                  │  │  query() → LLM API 调用     │   │
                  │  │  AsyncGenerator<Message>     │   │
                  │  └─────────────────────────────┘   │
                  └──────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────────┐
          ▼                ▼                    ▼
┌──────────────┐  ┌──────────────┐   ┌──────────────────┐
│  Local Agent │  │ Remote Agent │   │ In-Process       │
│  Task (同步/ │  │ Task (CCR    │   │ Teammate Task    │
│  异步后台)    │  │  远程沙箱)    │   │ (进程内协同)      │
└──────┬───────┘  └──────────────┘   └──────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────────────────┐
│                       Task 结果聚合                                │
│  ┌─────────────┐  ┌───────────────┐  ┌─────────────────────┐   │
│  │ task-       │  │ Agent Summary │  │ Notification        │   │
│  │ notification│  │ 进度摘要       │  │ (消息队列投递)       │   │
│  └─────────────┘  └───────────────┘  └─────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 完整调度链路

```
用户消息 → QueryEngine.submitMessage()
         → processUserInput() 解析/预处理
         → query() 进入 LLM 对话循环
         → LLM 返回 tool_use(Agent Tool)
         → AgentTool.call() 决策分支:
            ├─ team_name + name → spawnTeammate() (多Agent团队模式)
            ├─ isolation:remote → teleportToRemote() (远程沙箱)
            ├─ shouldRunAsync=true → registerAsyncAgent() + runAsyncAgentLifecycle()
            └─ 同步模式 → runAgent() 阻塞等待
         → 结果通过 <task-notification> XML 或 tool_result 返回
         → Coordinator/主 Agent 汇总结果
```

### 1.3 核心概念定义

| 概念 | 定义 | 源码位置 |
|------|------|----------|
| **Coordinator** | 编排者模式，负责任务分解、Worker 调度和结果综合 | `coordinator/coordinatorMode.ts` |
| **Worker** | 由 Coordinator 或主 Agent 通过 Agent Tool 派发的子 Agent | `tools/AgentTool/AgentTool.tsx` |
| **Teammate** | 通过 TeamCreateTool 创建的团队成员，拥有独立的 tmux 窗口/进程 | `tools/shared/spawnMultiAgent.ts` |
| **Fork Subagent** | 隐式分叉路径，子 Agent 继承父 Agent 完整对话上下文 | `tools/AgentTool/forkSubagent.ts` |
| **Task** | 统一的任务抽象，包含 7 种类型和完整生命周期管理 | `Task.ts` |

---

## 2. 核心抽象层

### 2.1 Task 类型系统（7 种 TaskType）

```typescript
// 文件: src/Task.ts
export type TaskType =
  | 'local_bash'          // 本地 Shell 命令（后台 Bash 执行）
  | 'local_agent'         // 本地 Agent（同步/异步子 Agent）
  | 'remote_agent'        // 远程 Agent（CCR 远程沙箱）
  | 'in_process_teammate' // 进程内 Teammate（AsyncLocalStorage 隔离）
  | 'local_workflow'      // 本地工作流任务
  | 'monitor_mcp'         // MCP 监控任务
  | 'dream'               // 自动记忆整合（AutoDream）
```

每种类型拥有独立的前缀 ID 生成策略，确保全局唯一：

```typescript
// 文件: src/Task.ts
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
};

export function generateTaskId(type: TaskType): string {
  const prefix = getTaskIdPrefix(type);
  const bytes = randomBytes(8);
  let id = prefix;
  for (let i = 0; i < 8; i++) {
    id += TASK_ID_ALPHABET[bytes[i]! % TASK_ID_ALPHABET.length];
  }
  return id; // 例: "a4k7x2m9p" — 36^8 ≈ 2.8万亿种组合
}
```

### 2.2 Task 接口与生命周期

```typescript
// 文件: src/Task.ts
export type TaskStatus = 'pending' | 'running' | 'completed' | 'failed' | 'killed';

export function isTerminalTaskStatus(status: TaskStatus): boolean {
  return status === 'completed' || status === 'failed' || status === 'killed';
}

export type TaskStateBase = {
  id: string;
  type: TaskType;
  status: TaskStatus;
  description: string;
  toolUseId?: string;
  startTime: number;
  endTime?: number;
  totalPausedMs?: number;
  outputFile: string;     // 磁盘输出路径（用于大结果转储）
  outputOffset: number;
  notified: boolean;       // 是否已发送 task-notification
};

export type Task = {
  name: string;
  type: TaskType;
  kill(taskId: string, setAppState: SetAppState): Promise<void>;
};
```

**生命周期状态机**:

```
                    ┌──────────┐
                    │ pending  │
                    └────┬─────┘
                         │ registerTask()
                         ▼
                    ┌──────────┐
           ┌───────│ running  │───────┐
           │       └────┬─────┘       │
           │            │             │
     kill()     complete/fail      kill()
           │            │             │
           ▼            ▼             ▼
    ┌──────────┐ ┌──────────┐  ┌──────────┐
    │  killed  │ │completed │  │  failed  │
    └──────────┘ └──────────┘  └──────────┘
         (终端状态，不再转换)
```

### 2.3 TaskState 联合类型

```typescript
// 文件: src/tasks/types.ts
export type TaskState =
  | LocalShellTaskState
  | LocalAgentTaskState
  | RemoteAgentTaskState
  | InProcessTeammateTaskState
  | LocalWorkflowTaskState
  | MonitorMcpTaskState
  | DreamTaskState;
```

通过 `getTaskByType(task.type)` 进行多态分发，每种 Task 类型注册自己的 `kill` 实现。

### 2.4 Coordinator 模式

```typescript
// 文件: src/coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE);
  }
  return false;
}
```

Coordinator 模式下，主 Agent 成为纯粹的编排者：
- 通过 `AgentTool` 派发 Worker
- 通过 `SendMessageTool` 继续已有 Worker
- 通过 `TaskStopTool` 停止 Worker
- 不直接执行代码操作

**核心系统提示词**（节选）:

```
You are Claude Code, an AI assistant that orchestrates software engineering
tasks across multiple workers.

## Your Role
You are a coordinator. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible
```

Coordinator 收到 Worker 结果通过 `<task-notification>` XML 消息:

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

---

## 3. Agent 工具架构

### 3.1 AgentTool 设计 — 入口、运行、Fork、Resume

AgentTool 是多 Agent 调度的核心入口，一个 Tool 同时支持 5 种运行模式：

```
AgentTool.call(input)
    │
    ├─ team_name + name → spawnTeammate() → TeammateSpawnedOutput
    │
    ├─ isolation: "remote" → teleportToRemote() → RemoteLaunchedOutput
    │
    ├─ isForkPath (无 subagent_type + fork gate on)
    │     → 继承父对话 + 父系统提示
    │     → shouldRunAsync=true → AsyncLaunchedOutput
    │
    ├─ shouldRunAsync=true (后台/Coordinator/forceAsync)
    │     → registerAsyncAgent()
    │     → void runAsyncAgentLifecycle()  (fire-and-forget)
    │     → AsyncLaunchedOutput
    │
    └─ 同步模式
          → registerAgentForeground()
          → runAgent() → AsyncGenerator<Message>
          → 可被 autoBackground 机制转为异步
          → CompletedOutput
```

**异步 Agent 启动关键代码**:

```typescript
// 文件: src/tools/AgentTool/AgentTool.tsx (节选)
if (shouldRunAsync) {
  const asyncAgentId = earlyAgentId;
  const agentBackgroundTask = registerAsyncAgent({
    agentId: asyncAgentId,
    description,
    prompt,
    selectedAgent,
    setAppState: rootSetAppState,
    toolUseId: toolUseContext.toolUseId,
  });

  // 注册名称映射（用于 SendMessage 路由）
  if (name) {
    rootSetAppState(prev => {
      const next = new Map(prev.agentNameRegistry);
      next.set(name, asAgentId(asyncAgentId));
      return { ...prev, agentNameRegistry: next };
    });
  }

  // fire-and-forget: 后台执行，不阻塞当前 turn
  void runWithAgentContext(asyncAgentContext, () =>
    wrapWithCwd(() => runAsyncAgentLifecycle({
      taskId: agentBackgroundTask.agentId,
      abortController: agentBackgroundTask.abortController!,
      makeStream: onCacheSafeParams => runAgent({ ... }),
      metadata,
      description,
      toolUseContext,
      rootSetAppState,
      agentIdForCleanup: asyncAgentId,
    }))
  );

  return { data: { status: 'async_launched', agentId, ... } };
}
```

### 3.2 Fork Subagent 机制

Fork 是一种隐式分叉策略——省略 `subagent_type` 时自动触发，子 Agent 继承父 Agent 的完整对话上下文和系统提示：

```typescript
// 文件: src/tools/AgentTool/forkSubagent.ts
export const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],
  maxTurns: 200,
  model: 'inherit',         // 继承父模型
  permissionMode: 'bubble', // 权限提示冒泡到父终端
  source: 'built-in',
  getSystemPrompt: () => '', // 实际使用父的 renderedSystemPrompt
} satisfies BuiltInAgentDefinition;
```

**Prompt Cache 共享策略**:

为了实现字节级相同的 API 请求前缀（最大化 prompt cache 命中），Fork 子 Agent 的消息构建极为精巧：

```typescript
// 文件: src/tools/AgentTool/forkSubagent.ts
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆父的完整 assistant 消息（所有 tool_use 块保持不变）
  const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID(), ... };

  // 2. 为每个 tool_use 生成相同的占位符 tool_result
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text', text: FORK_PLACEHOLDER_RESULT }], // 完全相同!
  }));

  // 3. 单条 user 消息：所有占位符结果 + 每个子 Agent 独有的指令
  const toolResultMessage = createUserMessage({
    content: [...toolResultBlocks, { type: 'text', text: buildChildMessage(directive) }],
  });

  return [fullAssistantMessage, toolResultMessage];
}
```

**结果**: 所有 Fork 子 Agent 共享相同的 `[history, assistant(all_tool_uses), user(placeholder_results...)]` 前缀，只有最后的 text block 不同 → 最大化 cache 命中率。

**递归分叉防护**:

```typescript
// 文件: src/tools/AgentTool/forkSubagent.ts
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false;
    return content.some(block =>
      block.type === 'text' && block.text.includes(`<${FORK_BOILERPLATE_TAG}>`)
    );
  });
}
```

### 3.3 Agent 记忆系统

Agent 支持三级持久化记忆（Memory Scope）:

```typescript
// 文件: src/tools/AgentTool/agentMemory.ts
export type AgentMemoryScope = 'user' | 'project' | 'local';

// 'user':   ~/.claude/agent-memory/<agentType>/MEMORY.md
// 'project': .claude/agent-memory/<agentType>/MEMORY.md
// 'local':  .claude/agent-memory-local/<agentType>/MEMORY.md

export function getAgentMemoryDir(agentType: string, scope: AgentMemoryScope): string {
  const dirName = sanitizeAgentTypeForPath(agentType);
  switch (scope) {
    case 'project': return join(getCwd(), '.claude', 'agent-memory', dirName) + sep;
    case 'local':   return getLocalAgentMemoryDir(dirName);
    case 'user':    return join(getMemoryBaseDir(), 'agent-memory', dirName) + sep;
  }
}
```

记忆在 Agent 启动时加载，通过 `buildMemoryPrompt()` 注入到系统提示中。

### 3.4 Resume 机制

已完成的 Agent 可以通过 `SendMessageTool` 恢复执行：

```typescript
// 文件: src/tools/AgentTool/resumeAgent.ts
export async function resumeAgentBackground({ agentId, prompt, ... }) {
  // 1. 从磁盘加载 transcript（完整对话历史）
  const [transcript, meta] = await Promise.all([
    getAgentTranscript(asAgentId(agentId)),
    readAgentMetadata(asAgentId(agentId)),
  ]);

  // 2. 过滤无效消息
  const resumedMessages = filterWhitespaceOnlyAssistantMessages(
    filterOrphanedThinkingOnlyMessages(filterUnresolvedToolUses(transcript.messages))
  );

  // 3. 重建 content replacement state（prompt cache 稳定性）
  const resumedReplacementState = reconstructForSubagentResume(...);

  // 4. 检测 worktree 是否仍存在
  const resumedWorktreePath = meta?.worktreePath ? await fsp.stat(...) : undefined;

  // 5. 重新注册为后台任务
  const agentBackgroundTask = registerAsyncAgent({ agentId, ... });

  // 6. 启动后台执行
  void runWithAgentContext(asyncAgentContext, () =>
    wrapWithCwd(() => runAsyncAgentLifecycle({
      makeStream: onCacheSafeParams => runAgent({
        promptMessages: [...resumedMessages, createUserMessage({ content: prompt })],
        contentReplacementState: resumedReplacementState,
        worktreePath: resumedWorktreePath,
        ...
      }),
    }))
  );
}
```

### 3.5 内置 Agent 类型分工

| Agent 类型 | 定位 | 工具访问 | 特点 |
|-----------|------|---------|------|
| **Explore** | 快速代码搜索 | 只读工具（Glob/Grep/Read/Bash） | 使用 haiku 模型，省略 CLAUDE.md 节省 token |
| **Plan** | 架构规划 | 只读工具（同 Explore） | 使用 inherit 模型，深度探索后输出实施计划 |
| **general-purpose** | 通用任务 | 全部工具 | 默认子 Agent，搜索+分析+执行 |
| **Fork** | 隐式分叉 | 继承父工具池 | 共享 prompt cache，bubble 权限模式 |

```typescript
// 文件: src/tools/AgentTool/built-in/exploreAgent.ts
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  disallowedTools: [AGENT_TOOL_NAME, EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME, FILE_WRITE_TOOL_NAME, NOTEBOOK_EDIT_TOOL_NAME],
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true, // 节省 5-15 Gtok/week
  getSystemPrompt: () => getExploreSystemPrompt(),
};
```

---

## 4. 任务调度机制

### 4.1 任务创建与分发

任务创建分为两条路径：

**路径 1: LocalAgentTask（子 Agent）**

```typescript
// 文件: src/tasks/LocalAgentTask/LocalAgentTask.tsx
export function registerAsyncAgent({ agentId, description, prompt, selectedAgent, setAppState, toolUseId }) {
  const id = agentId ?? generateTaskId('local_agent');
  const abortController = createAbortController();
  const task: LocalAgentTaskState = {
    ...createTaskStateBase(id, 'local_agent', description, toolUseId),
    status: 'running',
    prompt,
    abortController,
    progress: { toolUseCount: 0, tokenCount: 0 },
  };
  registerTask(task, setAppState);
  return { agentId: id, abortController };
}
```

**路径 2: InProcessTeammate（进程内 Teammate）**

```typescript
// 文件: src/tools/shared/spawnMultiAgent.ts
async function handleSpawnInProcess(input, context) {
  const config: InProcessSpawnConfig = {
    name: sanitizedName,
    teamName,
    prompt,
    color: teammateColor,
    planModeRequired: plan_mode_required ?? false,
    model,
  };

  const result = await spawnInProcessTeammate(config, context);

  // 启动执行循环（fire-and-forget）
  if (result.taskId && result.teammateContext) {
    startInProcessTeammate({
      identity: { agentId, agentName, teamName, ... },
      prompt,
      toolUseContext: { ...context, messages: [] },
      abortController: result.abortController,
    });
  }
}
```

### 4.2 并行执行模型

**多 Agent 并行启动**：在单条消息中发起多个 AgentTool 调用：

```
用户: "调查 auth bug"
Coordinator:
  AgentTool({ description: "调查 auth bug", subagent_type: "worker", prompt: "..." })
  AgentTool({ description: "研究安全 token 存储", subagent_type: "worker", prompt: "..." })
  → 两个 Worker 并行执行
```

**并发策略**（来自 Coordinator 系统提示）:
- 只读任务（研究）: 自由并行
- 写入任务（实现）: 同一文件集合一次一个
- 验证任务: 可在独立文件区域与实现并行

**Team 多 Agent 并行**: 每个 Teammate 运行在独立的 tmux pane 或 AsyncLocalStorage 隔离中：

```typescript
// 文件: src/tools/shared/spawnMultiAgent.ts
export async function spawnTeammate(config, context) {
  return handleSpawn(config, context);
  // → handleSpawn 检测后端:
  //   - isInProcessEnabled() → handleSpawnInProcess() (AsyncLocalStorage)
  //   - tmux/iTerm2 可用 → handleSpawnSplitPane() (tmux 分屏)
  //   - 兜底 → handleSpawnInProcess()
}
```

### 4.3 同步 → 异步自动转换

前台同步 Agent 在运行超过阈值时，可被自动转为后台异步：

```typescript
// 文件: src/tools/AgentTool/AgentTool.tsx
function getAutoBackgroundMs(): number {
  if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS) ||
      getFeatureValue('tengu_auto_background_agents', false)) {
    return 120_000; // 2 分钟后自动转后台
  }
  return 0;
}

// 在同步循环中 race between next message and background signal:
const raceResult = backgroundPromise
  ? await Promise.race([
      nextMessagePromise.then(r => ({ type: 'message', result: r })),
      backgroundPromise  // 超时后触发
    ])
  : { type: 'message', result: await nextMessagePromise };

if (raceResult.type === 'background' && foregroundTaskId) {
  // 转为后台执行
  void runWithAgentContext(syncAgentContext, async () => {
    for await (const msg of runAgent({ isAsync: true, ... })) {
      // 后台迭代
    }
    completeAsyncAgent(agentResult, rootSetAppState);
    enqueueAgentNotification({ status: 'completed', ... });
  });
  return { data: { status: 'async_launched', ... } };
}
```

### 4.4 结果聚合

Agent 完成后，结果通过 `enqueueAgentNotification` 投递到消息队列：

```typescript
// 文件: src/tools/AgentTool/agentToolUtils.ts (推断)
enqueueAgentNotification({
  taskId,
  description,
  status: 'completed',
  setAppState,
  finalMessage,     // Agent 最终文本输出
  usage: {
    totalTokens,
    toolUses,
    durationMs,
  },
  toolUseId,
  ...worktreeResult,
});
```

通知以 `<task-notification>` XML 格式作为 user-role 消息注入到 Coordinator 的下一条 turn 中。

### 4.5 停止与取消

```typescript
// 文件: src/tasks/stopTask.ts
export async function stopTask(taskId, context) {
  const task = appState.tasks?.[taskId];

  if (task.status !== 'running') {
    throw new StopTaskError(`Task ${taskId} is not running`, 'not_running');
  }

  const taskImpl = getTaskByType(task.type); // 多态分发
  await taskImpl.kill(taskId, setAppState);

  // Bash 任务: 抑制 "exit code 137" 噪音通知
  if (isLocalShellTask(task)) {
    setAppState(prev => ({
      ...prev,
      tasks: { ...prev.tasks, [taskId]: { ...prevTask, notified: true } },
    }));
  }
}
```

各 Task 类型的 kill 实现：

| Task 类型 | Kill 方式 | 源码 |
|-----------|----------|------|
| `local_agent` | `abortController.abort()` | `LocalAgentTask.tsx` |
| `remote_agent` | `archiveRemoteSession()` + 状态更新 | `RemoteAgentTask.tsx` |
| `in_process_teammate` | `killInProcessTeammate()` | `InProcessTeammateTask.tsx` |
| `local_bash` | `killTask()` (SIGTERM) | `LocalShellTask.tsx` |
| `dream` | `abortController.abort()` + `rollbackConsolidationLock()` | `DreamTask.ts` |

---

## 5. 关键代码实现

### 5.1 runAgent — Agent 执行核心

```typescript
// 文件: src/tools/AgentTool/runAgent.ts
export async function* runAgent({
  agentDefinition, promptMessages, toolUseContext, canUseTool, isAsync,
  querySource, override, model, availableTools, ...
}): AsyncGenerator<Message, void> {

  // 1. 解析模型
  const resolvedAgentModel = getAgentModel(
    agentDefinition.model, toolUseContext.options.mainLoopModel, model, permissionMode
  );

  // 2. 构建系统提示（Fork 继承父提示，普通路径重新构建）
  const agentSystemPrompt = override?.systemPrompt
    ? override.systemPrompt
    : asSystemPrompt(await getAgentSystemPrompt(agentDefinition, ...));

  // 3. 组装独立工具池（不受父工具限制影响）
  const resolvedTools = useExactTools
    ? availableTools
    : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools;

  // 4. 初始化 Agent 特有的 MCP 服务器
  const { clients, tools, cleanup } = await initializeAgentMcpServers(
    agentDefinition, toolUseContext.options.mcpClients
  );

  // 5. 执行 SubagentStart hooks
  for await (const hookResult of executeSubagentStartHooks(...)) {
    additionalContexts.push(...hookResult.additionalContexts);
  }

  // 6. 预加载 Agent frontmatter 中声明的 skills
  for (const skillName of agentDefinition.skills ?? []) {
    initialMessages.push(createUserMessage({ content: [...loadedSkillContent] }));
  }

  // 7. 进入 LLM 对话循环
  try {
    for await (const message of query({
      messages: initialMessages,
      systemPrompt: agentSystemPrompt,
      userContext, systemContext, canUseTool,
      toolUseContext: agentToolUseContext,
      querySource,
      maxTurns: maxTurns ?? agentDefinition.maxTurns,
    })) {
      if (isRecordableMessage(message)) {
        await recordSidechainTranscript([message], agentId, lastRecordedUuid);
        yield message;
      }
    }
  } finally {
    await mcpCleanup();        // 清理 Agent MCP 服务器
    clearSessionHooks(...);     // 清理会话 hooks
    killShellTasksForAgent(...);// 清理 Agent 创建的 shell 任务
  }
}
```

### 5.2 runAsyncAgentLifecycle — 异步 Agent 生命周期

```typescript
// 推断自 agentToolUtils.ts 的 runAsyncAgentLifecycle
export async function runAsyncAgentLifecycle({
  taskId, abortController, makeStream, metadata, description,
  toolUseContext, rootSetAppState, agentIdForCleanup, enableSummarization,
  getWorktreeResult,
}) {
  const agentMessages: Message[] = [];
  try {
    for await (const msg of makeStream(onCacheSafeParams)) {
      agentMessages.push(msg);
      // 更新进度追踪
      updateProgressFromMessage(tracker, msg, ...);
      updateAsyncAgentProgress(taskId, getProgressUpdate(tracker), rootSetAppState);
    }
    const agentResult = finalizeAgentTool(agentMessages, taskId, metadata);
    completeAsyncAgent(agentResult, rootSetAppState);  // 标记完成
    enqueueAgentNotification({ taskId, status: 'completed', ... });  // 通知父 Agent
  } catch (error) {
    if (error instanceof AbortError) {
      killAsyncAgent(taskId, rootSetAppState);
      enqueueAgentNotification({ taskId, status: 'killed', ... });
    } else {
      failAsyncAgent(taskId, errorMessage(error), rootSetAppState);
      enqueueAgentNotification({ taskId, status: 'failed', ... });
    }
  } finally {
    clearInvokedSkillsForAgent(agentIdForCleanup);
    clearDumpState(agentIdForCleanup);
    await getWorktreeResult?.();  // 清理 worktree
  }
}
```

### 5.3 Agent 进度追踪

```typescript
// 文件: src/tasks/LocalAgentTask/LocalAgentTask.tsx
export type ProgressTracker = {
  toolUseCount: number;
  latestInputTokens: number;       // input_tokens 是累积的，保留最新值
  cumulativeOutputTokens: number;  // output_tokens 是每次的，累加
  recentActivities: ToolActivity[];
};

export function updateProgressFromMessage(tracker, message, resolveActivityDescription, tools) {
  if (message.type !== 'assistant') return;
  const usage = message.message.usage;
  tracker.latestInputTokens = usage.input_tokens
    + (usage.cache_creation_input_tokens ?? 0)
    + (usage.cache_read_input_tokens ?? 0);
  tracker.cumulativeOutputTokens += usage.output_tokens;

  for (const content of message.message.content) {
    if (content.type === 'tool_use') {
      tracker.toolUseCount++;
      tracker.recentActivities.push({
        toolName: content.name,
        input: content.input,
        activityDescription: resolveActivityDescription?.(content.name, content.input),
      });
    }
  }
  // 保持最近 5 条活动记录
  while (tracker.recentActivities.length > MAX_RECENT_ACTIVITIES) {
    tracker.recentActivities.shift();
  }
}
```

### 5.4 Worktree 隔离

Agent 可在独立 Git Worktree 中执行，实现文件系统隔离：

```typescript
// 文件: src/tools/AgentTool/AgentTool.tsx (节选)
if (effectiveIsolation === 'worktree') {
  const slug = `agent-${earlyAgentId.slice(0, 8)}`;
  worktreeInfo = await createAgentWorktree(slug);
}

// 清理逻辑：如果 Agent 没有改动则自动删除 worktree
const cleanupWorktreeIfNeeded = async () => {
  if (!worktreeInfo) return {};
  if (headCommit) {
    const changed = await hasWorktreeChanges(worktreePath, headCommit);
    if (!changed) {
      await removeAgentWorktree(worktreePath, worktreeBranch, gitRoot);
      return {};
    }
  }
  return { worktreePath, worktreeBranch };
};
```

### 5.5 Teammate Mailbox 通信

```typescript
// 文件: src/tools/shared/spawnMultiAgent.ts
// 初始指令通过 mailbox 发送
await writeToMailbox(sanitizedName, {
  from: TEAM_LEAD_NAME,
  text: prompt,
  timestamp: new Date().toISOString(),
}, teamName);
```

### 5.6 Dream 任务（自动记忆整合）

```typescript
// 文件: src/tasks/DreamTask/DreamTask.ts
export type DreamTaskState = TaskStateBase & {
  type: 'dream';
  phase: DreamPhase;            // 'starting' | 'updating'
  sessionsReviewing: number;
  filesTouched: string[];
  turns: DreamTurn[];
  abortController?: AbortController;
  priorMtime: number;           // 用于回滚 consolidation lock
};
```

Dream 是后台自动运行的记忆整合子 Agent，定期扫描会话历史并整合记忆。

### 5.7 Task 通知投递

```typescript
// 文件: src/tasks/LocalAgentTask/LocalAgentTask.tsx (推断)
export function enqueueAgentNotification({ taskId, description, status, setAppState, finalMessage, usage, toolUseId }) {
  const message = `<${TASK_NOTIFICATION_TAG}>
<${TASK_ID_TAG}>${taskId}</${TASK_ID_TAG}>
<${STATUS_TAG}>${status}</${STATUS_TAG}>
<${SUMMARY_TAG}>${escapeXml(summary)}</${SUMMARY_TAG}>
${finalMessage ? `<result>${escapeXml(finalMessage)}</result>` : ''}
${usage ? `<usage><total_tokens>${usage.totalTokens}</total_tokens>
<tool_uses>${usage.toolUses}</tool_uses>
<duration_ms>${usage.durationMs}</duration_ms></usage>` : ''}
</${TASK_NOTIFICATION_TAG}>`;

  enqueuePendingNotification({ value: message, mode: 'task-notification', priority: 'next' });
}
```

### 5.8 Shell Stall Watchdog

后台 Shell 任务自动检测交互式阻塞：

```typescript
// 文件: src/tasks/LocalShellTask/LocalShellTask.tsx
const PROMPT_PATTERNS = [
  /\(y\/n\)/i, /\[y\/n\]/i, /\(yes\/no\)/i,
  /\b(?:Do you|Would you|Shall I|Are you sure|Ready to)\b.*\? *$/i,
  /Press (any key|Enter)/i, /Continue\?/i, /Overwrite\?/i,
];

function startStallWatchdog(taskId, description, kind, toolUseId, agentId) {
  const timer = setInterval(() => {
    // 检查输出是否 45 秒无增长
    if (Date.now() - lastGrowth >= STALL_THRESHOLD_MS) {
      // 检查最后输出是否像交互式提示
      if (looksLikePrompt(tailContent)) {
        // 发送通知: "command appears to be waiting for interactive input"
        enqueuePendingNotification({ ... });
      }
    }
  }, STALL_CHECK_INTERVAL_MS);
}
```

---

## 6. 设计模式提炼

### 6.1 策略模式 (Strategy Pattern)

**Task 类型的多态分发**:

```typescript
// 文件: src/tasks/stopTask.ts
const taskImpl = getTaskByType(task.type); // 根据类型获取策略实现
await taskImpl.kill(taskId, setAppState);  // 统一接口，不同实现
```

每种 TaskType 注册自己的 `Task` 实现，`getTaskByType` 返回对应策略。

### 6.2 异步生成器模式 (Async Generator Pattern)

**runAgent 返回 `AsyncGenerator<Message>`**:

```typescript
// 文件: src/tools/AgentTool/runAgent.ts
export async function* runAgent({ ... }): AsyncGenerator<Message, void> {
  for await (const message of query({ ... })) {
    if (isRecordableMessage(message)) {
      yield message; // 流式产出消息
    }
  }
}
```

消费端通过 `for await...of` 逐条处理，支持：
- 同步消费（阻塞等待）
- 后台消费（fire-and-forget）
- 自动转后台（Promise.race）

### 6.3 观察者模式 (Observer Pattern)

**任务通知系统**:

```
Agent 完成 → enqueueAgentNotification()
           → enqueuePendingNotification()
           → 作为 user-role 消息注入下一 turn
           → Coordinator 的 query() 循环读取
```

### 6.4 模板方法模式 (Template Method)

**Agent 定义统一接口，内置 Agent 和自定义 Agent 共享骨架**:

```typescript
type BuiltInAgentDefinition = {
  agentType: string;
  whenToUse: string;
  tools?: string[];
  disallowedTools?: string[];
  model?: string;
  permissionMode?: PermissionMode;
  getSystemPrompt: (ctx: { toolUseContext: ToolUseContext }) => string;
  // ...
};
```

### 6.5 Fire-and-Forget 模式

异步 Agent 启动后不等待结果，通过 `void` 关键字标记：

```typescript
// 文件: src/tools/AgentTool/AgentTool.tsx
void runWithAgentContext(asyncAgentContext, () =>
  wrapWithCwd(() => runAsyncAgentLifecycle({ ... }))
);
// 立即返回 AsyncLaunchedOutput，结果通过 notification 回传
```

### 6.6 多态注册表 (Polymorphic Registry)

```typescript
// 文件: src/tasks/RemoteAgentTask/RemoteAgentTask.tsx
const completionCheckers = new Map<RemoteTaskType, RemoteTaskCompletionChecker>();

export function registerCompletionChecker(remoteTaskType, checker) {
  completionCheckers.set(remoteTaskType, checker);
}
```

### 6.7 权限冒泡 (Bubble Permission)

Fork 子 Agent 使用 `permissionMode: 'bubble'`，将权限提示冒泡到父 Agent 的终端：

```typescript
// 子 Agent 不直接弹权限提示，而是冒泡到父终端让用户处理
const shouldAvoidPrompts = agentPermissionMode === 'bubble' ? false : isAsync;
```

---

## 7. 可借鉴方案

### 7.1 统一的 Task 抽象

Claude Code 将所有异步操作（Shell、Agent、Teammate、Dream）统一为 `TaskState` 联合类型，共享：
- 统一的状态机（pending → running → completed/failed/killed）
- 统一的注册/更新/通知机制（`registerTask`, `updateTaskState`, `enqueuePendingNotification`）
- 统一的 UI 展示（后台任务面板、进度条）

**借鉴**: 任何多 Agent 系统都应建立统一的任务抽象，而非为每种 Agent 类型实现独立的管理逻辑。

### 7.2 Prompt Cache 友好的消息设计

Fork 机制展示了如何通过精心的消息结构设计最大化 API prompt cache 命中：
- 所有子 Agent 共享相同的消息前缀
- 只有最后一条 user message 的 text block 不同
- 系统提示通过 `renderedSystemPrompt` 直接传递已渲染的字节，避免重新构建导致的偏差

**借鉴**: 在多 Agent 并行场景中，消息结构设计直接影响 API 成本和延迟。

### 7.3 同步/异步透明转换

Agent 可以在运行中从同步模式无缝转为异步（后台）模式，通过 `Promise.race` 和 `registerAgentForeground` 实现：

```typescript
const raceResult = await Promise.race([
  nextMessagePromise.then(r => ({ type: 'message', result: r })),
  backgroundPromise,  // 2分钟超时触发
]);
```

**借鉴**: 长时间运行的子任务应支持动态切换执行模式，提升用户体验。

### 7.4 分层权限模型

- **Worker**: 独立权限模式（`acceptEdits`），不受父限制
- **Fork**: 冒泡模式（`bubble`），权限提示转发给父终端
- **Teammate**: 可配置 `plan_mode_required`，实施计划审批流程
- **Explore/Plan**: 只读，禁止文件修改

**借鉴**: 不同角色需要不同的权限策略，硬编码统一权限会限制系统灵活性。

### 7.5 Worktree 隔离

通过 Git Worktree 实现文件系统级隔离：
- 自动创建、自动清理（无改动时删除）
- 有改动时保留，返回 worktree 路径和分支信息

**借鉴**: 多 Agent 并行写文件时，需要某种形式的隔离（文件级、目录级、进程级），Git Worktree 是一个低成本的方案。

### 7.6 Agent 记忆三级分离

- `user` scope: 跨项目通用知识
- `project` scope: 项目特定知识（可提交到 VCS 共享）
- `local` scope: 本机特定知识（不入 VCS）

**借鉴**: Agent 记忆应区分作用域，避免全局记忆污染或丢失项目特定上下文。

---

## 8. 总结

Claude Code 的多 Agent 并行调度架构是一个精心设计的系统，核心特点包括：

1. **统一的 Task 抽象**: 7 种 TaskType 共享状态机和生命周期管理，通过多态分发实现差异化行为

2. **灵活的执行模型**: 支持同步、异步、Coordinator 编排、Fork 分叉、远程执行 5 种模式，且可动态转换

3. **精巧的 Prompt Cache 策略**: Fork 路径通过继承父对话和系统提示、使用占位符结果，实现字节级相同的 API 请求前缀

4. **分层隔离设计**: 从进程内 AsyncLocalStorage 到 Git Worktree 到远程 CCR 沙箱，提供不同级别的隔离

5. **完备的生命周期管理**: 从创建、进度追踪、自动转后台、完成/失败/取消通知，到 worktree 清理和 hook 注销

6. **角色化分工**: Explore（搜索）、Plan（规划）、Worker（执行）、Coordinator（编排）各司其职，通过工具限制和系统提示精确控制行为边界

这套架构为构建生产级多 Agent 系统提供了可复用的设计范式，尤其在任务抽象、消息设计、权限控制和资源隔离方面具有很高的参考价值。
