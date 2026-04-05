# hooks/ 模块深度分析

## 模块概览

| 指标 | 值 |
|------|-----|
| 文件数量 | 104 |
| 核心职责 | React Hooks 层：权限交互、UI 状态管理、工具执行控制、通知系统、输入处理 |
| 顶层文件 | 80+ 独立 hook 文件 |
| 子目录 | `toolPermission/` (权限处理), `notifs/` (通知) |

### 关键文件列表

| Hook | 核心职责 |
|------|----------|
| `useCanUseTool.tsx` | **核心**：工具权限决策入口 |
| `useSettings.ts` | 响应式设置访问 |
| `useMainLoopModel.ts` | 当前模型响应式获取 |
| `useVirtualScroll.ts` | 高性能虚拟滚动 |
| `useIDEIntegration.tsx` | IDE 集成生命周期 |
| `useSwarmInitialization.ts` | 多 Agent 初始化 |
| `useQueueProcessor.ts` | 命令队列处理 |
| `toolPermission/` | 权限决策三路分发 |
| `notifs/` | 15+ 种通知 hook |
| `useTypeahead.tsx` | 命令/文件自动补全 |

---

## 架构总览

```
hooks/
├── 权限决策层
│   ├── useCanUseTool.tsx          ← 工具权限入口
│   └── toolPermission/
│       ├── PermissionContext.ts   ← 权限上下文工厂
│       ├── permissionLogging.ts  ← 权限审计日志
│       └── handlers/
│           ├── interactiveHandler.ts  ← 主 Agent 交互式权限
│           ├── coordinatorHandler.ts  ← 协调器模式权限
│           └── swarmWorkerHandler.ts  ← Swarm Worker 权限
│
├── UI 状态层
│   ├── useSettings.ts            ← 响应式设置
│   ├── useMainLoopModel.ts       ← 响应式模型
│   ├── useVirtualScroll.ts       ← 虚拟滚动引擎
│   ├── useTerminalSize.ts        ← 终端尺寸
│   ├── useAfterFirstRender.ts    ← 延迟副作用
│   └── useDeferredHookMessages.ts ← 延迟 Hook 消息
│
├── 输入处理层
│   ├── useTypeahead.tsx          ← 自动补全
│   ├── useInputBuffer.ts         ← 输入缓冲
│   ├── useVimInput.ts            ← Vim 模式输入
│   ├── usePasteHandler.ts        ← 粘贴处理
│   ├── useArrowKeyHistory.tsx    ← 方向键历史
│   ├── useSearchInput.ts         ← 搜索输入
│   └── useHistorySearch.ts       ← 历史搜索
│
├── 通信/集成层
│   ├── useIDEIntegration.tsx     ← IDE 集成
│   ├── useSwarmInitialization.ts ← Swarm 初始化
│   ├── useReplBridge.tsx         ← REPL 桥接
│   ├── useMailboxBridge.ts       ← 邮箱桥接
│   ├── useMergedClients.ts       ← MCP 客户端合并
│   ├── useMergedTools.ts         ← 工具合并
│   ├── useMergedCommands.ts      ← 命令合并
│   └── useDirectConnect.ts       ← 直连
│
├── 通知系统 (notifs/)
│   ├── useStartupNotification.ts
│   ├── useSettingsErrors.tsx
│   ├── useMcpConnectivityStatus.tsx
│   ├── useRateLimitWarningNotification.tsx
│   ├── useFastModeNotification.tsx
│   └── ... (共 15+ 通知 hook)
│
└── 辅助工具层
    ├── useTimeout.ts             ← 定时器
    ├── useDoublePress.ts         ← 双击检测
    ├── useBlink.ts               ← 光标闪烁
    ├── useElapsedTime.ts         ← 计时器
    ├── useMemoryUsage.ts         ← 内存监控
    ├── useCopyOnSelect.ts        ← 选中复制
    └── useVoice.ts / useVoiceEnabled.ts ← 语音控制
```

---

## 1. 权限决策系统 (`useCanUseTool` + `toolPermission/`)

### 1.1 架构设计

这是 hooks 模块中最核心的部分，实现了工具调用的权限决策流程。

```
useCanUseTool (入口)
    │
    ├─ 1. 创建 PermissionContext
    │      ├─ 工具 + 输入 + 上下文
    │      └─ 日志、持久化、取消/中止方法
    │
    ├─ 2. hasPermissionsToUseTool() 规则检查
    │      ├─ allow → 直接通过
    │      └─ deny → 直接拒绝
    │
    ├─ 3. Classifier 检查 (auto mode)
    │      └─ YOLO 白名单 → 自动通过
    │
    └─ 4. 三路分发 (behavior === "ask")
           ├─ handleCoordinatorPermission  ← 协调器模式
           ├─ handleInteractivePermission  ← 交互模式（主）
           └─ handleSwarmWorkerPermission  ← Swarm Worker
```

### 1.2 useCanUseTool 实现

```typescript
// useCanUseTool.tsx - 核心权限 hook
function useCanUseTool(setToolUseConfirmQueue, setToolPermissionContext) {
  return async (tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision) => {
    return new Promise(resolve => {
      // 1. 创建权限上下文
      const ctx = createPermissionContext(
        tool, input, toolUseContext, assistantMessage, toolUseID,
        setToolPermissionContext,
        createPermissionQueueOps(setToolUseConfirmQueue)
      )

      // 2. 获取决策（强制决策 or 规则检查）
      const decisionPromise = forceDecision !== undefined
        ? Promise.resolve(forceDecision)
        : hasPermissionsToUseTool(tool, input, toolUseContext, assistantMessage, toolUseID)

      decisionPromise.then(async result => {
        if (result.behavior === 'allow') {
          // 分类器自动通过
          if (result.decisionReason?.classifier === 'auto-mode') {
            setYoloClassifierApproval(toolUseID, result.decisionReason.reason)
          }
          resolve(ctx.buildAllow(result.updatedInput ?? input))
          return
        }

        // 3. 获取工具描述用于 UI
        const description = await tool.description(input, { ... })

        switch (result.behavior) {
          case 'deny':    // 直接拒绝
          case 'ask':     // 推送到交互流程
        }
      })
    })
  }
}
```

### 1.3 PermissionContext 工厂

`toolPermission/PermissionContext.ts` 是权限决策的"瑞士军刀"，提供了完整的权限操作接口：

```typescript
function createPermissionContext(tool, input, toolUseContext, ...) {
  return Object.freeze({
    // 日志方法
    logDecision(args) { logPermissionDecision(...) },
    logCancelled() { logEvent('tengu_tool_use_cancelled', ...) },

    // 权限持久化
    async persistPermissions(updates) {
      persistPermissionUpdates(updates)
      const appState = toolUseContext.getAppState()
      setToolPermissionContext(applyPermissionUpdates(appState, updates))
    },

    // 中止检查
    resolveIfAborted(resolve) { ... },

    // 取消并中止
    cancelAndAbort(feedback) {
      toolUseContext.abortController.abort()
      return { behavior: 'ask', message: REJECT_MESSAGE }
    },

    // Hook 执行
    async runHooks(permissionMode, suggestions) {
      for await (const hookResult of executePermissionRequestHooks(...)) { ... }
    },

    // 构建结果
    buildAllow(updatedInput) { ... },
    buildDeny(message, reason) { ... },

    // 队列操作
    pushToQueue(item) { ... },
    removeFromQueue() { ... },
  })
}
```

**关键设计 - ResolveOnce**：

```typescript
// 防止权限 Promise 被多次 resolve
function createResolveOnce<T>(resolve) {
  let claimed = false
  return {
    resolve(value) { if (!delivered) { delivered = true; claimed = true; resolve(value) } },
    claim() { if (claimed) return false; claimed = true; return true },
    isResolved() { return claimed },
  }
}
```

### 1.4 交互式权限处理器

`toolPermission/handlers/interactiveHandler.ts` (~540 行) 是最复杂的权限处理器：

```typescript
function handleInteractivePermission(params, resolve) {
  const { resolve: resolveOnce, isResolved, claim } = createResolveOnce(resolve)
  let userInteracted = false

  // 1. 推送 ToolUseConfirm 到确认队列
  // 2. 异步运行权限 Hook 和 bash 分类器
  // 3. 用户交互与自动检查竞争
  //    - 用户先操作 → 取消自动检查
  //    - 自动检查先完成 → 更新 UI 状态
}
```

核心特性：
- **竞争解决**：用户操作与自动分类器结果竞争，先到先得
- **Bridge 权限**：支持远程客户端（如手机）审批
- **Channel 权限**：MCP 通道权限请求
- **Checkmark 过渡**：自动审批后有短暂的 checkmark 显示

### 1.5 三种 Handler 的场景

| Handler | 场景 | 特点 |
|---------|------|------|
| `interactiveHandler` | 主 Agent 直接与用户交互 | 推送 UI 队列，等待用户操作 |
| `coordinatorHandler` | 协调器 Agent | 通过 mailbox 传递权限请求 |
| `swarmWorkerHandler` | Swarm Worker Agent | 通过 leader 权限桥接 |

---

## 2. UI 状态 Hooks

### 2.1 useSettings

极简但关键的 hook，连接 AppState 和 React 响应式：

```typescript
export function useSettings(): ReadonlySettings {
  return useAppState(s => s.settings)
}
```

当配置文件在磁盘上变更时，`settingsChangeDetector` 自动触发 AppState 更新，进而触发所有使用 `useSettings()` 的组件重渲染。

### 2.2 useMainLoopModel

```typescript
export function useMainLoopModel(): ModelName {
  const mainLoopModel = useAppState(s => s.mainLoopModel)
  const mainLoopModelForSession = useAppState(s => s.mainLoopModelForSession)

  // GrowthBook 刷新时强制重渲染
  // 因为 model alias 解析依赖 GB 远程配置
  const [, forceRerender] = useReducer(x => x + 1, 0)
  useEffect(() => onGrowthBookRefresh(forceRerender), [])

  return parseUserSpecifiedModel(
    mainLoopModelForSession ?? mainLoopModel ?? getDefaultMainLoopModelSetting()
  )
}
```

**设计亮点**：模型 alias（如 "sonnet"）需要远程配置才能解析为具体模型名。通过 `onGrowthBookRefresh` 订阅，确保远程配置加载完成后组件正确更新。

### 2.3 useVirtualScroll

约 730 行，是终端 UI 性能的关键：

```typescript
const DEFAULT_ESTIMATE = 3      // 未测量项的估计高度（行）
const OVERSCAN_ROWS = 80        // 视口上下额外渲染的行数
const COLD_START_COUNT = 30     // 初始渲染项数
const SCROLL_QUANTUM = 40       // 滚动量化（减少重渲染频率）
const PESSIMISTIC_HEIGHT = 1    // 未测量项的最坏假设
const MAX_MOUNTED_ITEMS = 300   // 最大挂载项数
```

**核心优化策略**：
- **滚动量化**：不是每次 wheel tick 都触发 React commit，而是量化到 OVERSCAN_ROWS/2
- **悲观高度估计**：假设未测量项高度为 1 行，确保视口完全覆盖
- **useSyncExternalStore**：直接订阅 ScrollBox 的 scrollTop 变化，绕过 React context 延迟

---

## 3. 输入处理 Hooks

### 3.1 useTypeahead

命令/文件自动补全系统，支持：
- 斜杠命令补全
- 文件路径补全
- Shell 历史补全

### 3.2 useInputBuffer

管理用户输入缓冲区，处理特殊字符和组合键。

### 3.3 useVimInput

实现 Vim 模式的输入处理（普通模式、插入模式、命令模式）。

### 3.4 useArrowKeyHistory

```typescript
// 方向键历史导航
export type HistoryMode = 'search' | 'navigation'
// 支持两种模式：
// - search: 在历史中搜索匹配
// - navigation: 上下浏览历史
```

---

## 4. 通信/集成 Hooks

### 4.1 useIDEIntegration

```typescript
// IDE 集成生命周期管理
export function useIDEIntegration({
  autoConnectIdeFlag,
  ideToInstallExtension,
  setDynamicMcpConfig,
  setShowIdeOnboarding,
  setIDEInstallationState,
}) {
  // 自动检测 IDE 并连接
  // 通过 MCP 协议与 IDE 通信
  // 支持 SSE 和 WebSocket 两种连接方式
}
```

### 4.2 useSwarmInitialization

```typescript
// Swarm 多 Agent 初始化
export function useSwarmInitialization(setAppState, initialMessages, { enabled = true }) {
  useEffect(() => {
    if (!enabled || !isAgentSwarmsEnabled()) return

    // 处理恢复的会话（从 --resume 或 /resume）
    // 从 transcript 消息中提取 teamName/agentName
    // 或从环境变量读取新会话上下文
  }, [enabled])
}
```

### 4.3 useQueueProcessor

```typescript
// 统一命令队列处理
export function useQueueProcessor({ executeQueuedInput, hasActiveLocalJsxUI, queryGuard }) {
  const isQueryActive = useSyncExternalStore(queryGuard.subscribe, queryGuard.getSnapshot)
  const queueSnapshot = useSyncExternalStore(subscribeToCommandQueue, getCommandQueueSnapshot)

  useEffect(() => {
    if (isQueryActive || hasActiveLocalJsxUI) return
    processQueueIfReady()
  }, [isQueryActive, hasActiveLocalJsxUI, queueSnapshot])
}
```

**优先级**：`now` > `next`（用户输入） > `later`（任务通知）

### 4.4 useMergedClients / useMergedTools / useMergedCommands

这三个 hook 负责合并多个来源的 MCP 客户端、工具和命令：
- 内置工具
- 插件提供的工具
- MCP 服务器提供的工具
- Agent 定义的命令

---

## 5. 通知系统 (`notifs/`)

### 5.1 通知 Hook 列表

| Hook | 触发条件 |
|------|----------|
| `useStartupNotification` | 启动时 |
| `useSettingsErrors` | 配置错误 |
| `useMcpConnectivityStatus` | MCP 连接状态变化 |
| `useRateLimitWarningNotification` | 速率限制 |
| `useFastModeNotification` | Fast mode 状态 |
| `useDeprecationWarningNotification` | 弃用警告 |
| `useNpmDeprecationNotification` | NPM 弃用 |
| `useAutoModeUnavailableNotification` | Auto mode 不可用 |
| `useModelMigrationNotifications` | 模型迁移 |
| `useLspInitializationNotification` | LSP 初始化 |
| `usePluginInstallationStatus` | 插件安装状态 |
| `usePluginAutoupdateNotification` | 插件自动更新 |
| `useTeammateShutdownNotification` | Teammate 关闭 |
| `useIDEStatusIndicator` | IDE 状态指示 |
| `useInstallMessages` | 安装消息 |

---

## 6. 其他重要 Hooks

### 会话/任务管理

| Hook | 职责 |
|------|------|
| `useRemoteSession` | 远程会话管理 |
| `useSSHSession` | SSH 会话 |
| `useTasksV2` | 任务系统 V2 |
| `useScheduledTasks` | 定时任务 |
| `useTaskListWatcher` | 任务列表监听 |
| `useInboxPoller` | 收件箱轮询 |

### Agent/Swarm

| Hook | 职责 |
|------|------|
| `useSwarmPermissionPoller` | Swarm 权限轮询 |
| `useTeammateViewAutoExit` | Teammate 视图自动退出 |

### UI 辅助

| Hook | 职责 |
|------|------|
| `useBlink` | 光标闪烁 |
| `useDoublePress` | 双击检测 |
| `useMinDisplayTime` | 最小显示时间 |
| `useNotifyAfterTimeout` | 超时通知 |
| `useCopyOnSelect` | 选中复制 |
| `useDiffData` / `useTurnDiffs` | Diff 数据 |
| `useDiffInIDE` | IDE 中显示 Diff |

### 认证/安全

| Hook | 职责 |
|------|------|
| `useApiKeyVerification` | API Key 验证 |
| `useExitOnCtrlCD` | Ctrl+C 退出 |
| `useExitOnCtrlCDWithKeybindings` | 带键绑定的退出 |

---

## Hook 与 Context 的配合模式

Claude Code 的 hooks 模块遵循以下模式：

### 模式 1：AppState → useAppState → Hook

```
AppState (全局状态)
    ↓ useAppState(selector)
  Hook (计算/副作用)
    ↓ useCallback/useEffect
  UI 组件
```

大多数 hook 通过 `useAppState` 订阅全局状态变更。

### 模式 2：useSyncExternalStore 订阅外部存储

```
外部 Store (Module-level Map/Queue)
    ↓ useSyncExternalStore(subscribe, getSnapshot)
  Hook
    ↓
  UI 组件
```

用于 `useQueueProcessor`、`useVirtualScroll` 等需要直接订阅非 React 状态的场景。

### 模式 3：Callback → setState → Hook

```
事件 (用户操作/API 响应)
    ↓ setState
  Hook (useCanUseTool)
    ↓ setToolUseConfirmQueue
  UI 组件 (PermissionRequest)
```

权限系统使用这种回调模式，将 Promise resolve 存入闭包。

---

## 设计优缺点

### 优点
1. **单一职责**：每个 hook 职责明确，文件命名直观
2. **类型安全**：`useCanUseTool` 导出 `CanUseToolFn` 类型，确保调用类型正确
3. **竞争安全**：`createResolveOnce` 防止权限 Promise 多次 resolve
4. **性能优化**：`useVirtualScroll` 的滚动量化、`useMainLoopModel` 的 GB 刷新订阅
5. **模块化通知**：每种通知独立 hook，可按需加载

### 缺点
1. **文件数量过多**：104 个文件增加了导航成本
2. **权限系统复杂**：`useCanUseTool` + 3 个 handler + PermissionContext，调用链深
3. **React Compiler 依赖**：`_c(n)` 编译器运行时调用出现在多个文件中，增加了阅读难度
4. **prop drilling**：`setToolUseConfirmQueue` 通过参数传递而非 context
5. **缺少统一错误边界**：各 hook 自行处理错误，缺少全局 hook 错误处理
