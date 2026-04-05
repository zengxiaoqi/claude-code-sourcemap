# 设计模式：Git Worktree 隔离

## 1. 模式概述

**问题**：多个 Agent 并行工作时，如果同时修改同一文件，会产生冲突。即使修改不同文件，Git 状态也会混乱。

**解决方案**：使用 Git Worktree 为每个 Agent 创建独立的文件系统视图——同一仓库的不同工作目录，共享 .git 目录但独立工作区。

**隔离层级图**：

```
┌─────────────────────────────────────────────────────────────┐
│                      隔离层级金字塔                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 4: 容器级隔离                                         │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Remote Agent (CCR 沙箱)                               │ │
│  │  完全独立的容器环境，最高隔离级别                          │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  Level 3: 进程级隔离                                         │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Local Shell Task                                      │ │
│  │  独立进程，共享文件系统                                   │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  Level 2: 文件系统级隔离（Worktree）                          │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Fork Agent with Worktree                              │ │
│  │  独立工作目录，共享 Git 历史                              │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  Level 1: 状态级隔离（AsyncLocalStorage）                     │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  In-Process Teammate                                   │ │
│  │  同进程，独立对话上下文                                   │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  Level 0: 无隔离                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  Fork Agent (无 Worktree)                              │ │
│  │  共享工作目录和 Git 状态                                  │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 源码实现分析

### 2.1 Worktree 创建流程

> 文件：`tools/AgentTool/forkSubagent.ts`

Worktree 的使用与 Fork 机制紧密绑定。当 Fork Agent 需要修改文件时，可以选择在隔离的 Worktree 中执行：

```typescript
/**
 * Notice injected into fork children running in an isolated worktree.
 * Tells the child to translate paths from the inherited context, re-read
 * potentially stale files, and that its changes are isolated.
 */
export function buildWorktreeNotice(
  parentCwd: string,
  worktreeCwd: string,
): string {
  return `You've inherited the conversation context above from a parent agent working in ${parentCwd}. You are operating in an isolated git worktree at ${worktreeCwd} — same repository, same relative file structure, separate working copy. Paths in the inherited context refer to the parent's working directory; translate them to your worktree root. Re-read files before editing if the parent may have modified them since they appear in the context. Your changes stay in this worktree and will not affect the parent's files.`
}
```

### 2.2 Worktree 目录结构

```
主工作目录：/project/my-app/
    ├── .git/               ← 共享的 Git 目录
    ├── src/
    │   └── main.ts
    └── package.json

Worktree 1：/project/my-app-worktrees/agent-explore/
    ├── .git                ← 指向主 .git 的文件（非目录）
    ├── src/
    │   └── main.ts         ← 独立的工作区副本
    └── package.json

Worktree 2：/project/my-app-worktrees/agent-implement/
    ├── .git                ← 指向主 .git 的文件
    ├── src/
    │   └── main.ts         ← 独立的工作区副本
    └── package.json
```

### 2.3 Scratchpad 机制

> 文件：`coordinator/coordinatorMode.ts`

除了 Worktree 隔离，Claude Code 还提供了 Scratchpad 目录用于跨 Worker 的知识共享：

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  // ...

  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\n\nScratchpad directory: ${scratchpadDir}
Workers can read and write here without permission prompts. Use this for durable cross-worker knowledge — structure files however fits the work.`
  }

  return { workerToolsContext: content }
}
```

**Scratchpad 用途**：
- 跨 Worker 共享研究发现
- 存储中间结果
- 协调多 Agent 的工作
- 文件结构由 Agent 自行决定

### 2.4 Worktree 清理机制

> 文件：`tasks/LocalAgentTask/` 等

Worktree 的生命周期管理遵循以下规则：

1. **无改动时自动清理**：
   - Agent 完成任务后检查是否有未提交的改动
   - 如果没有，自动删除 Worktree

2. **有改动时保留**：
   - 如果有改动，保留 Worktree
   - 返回 Worktree 路径和分支信息给用户
   - 用户可以手动审查和合并

3. **任务终止时清理**：
   - 用户取消任务时，如果有未保存改动，提示用户
   - 确认后清理 Worktree

```typescript
// 伪代码示意
async function cleanupWorktree(worktreePath: string): Promise<CleanupResult> {
  const hasChanges = await checkGitStatus(worktreePath)
  
  if (!hasChanges) {
    await deleteWorktree(worktreePath)
    return { status: 'deleted' }
  }
  
  const branchName = await getCurrentBranch(worktreePath)
  return { 
    status: 'preserved',
    path: worktreePath,
    branch: branchName,
    message: 'Worktree preserved due to uncommitted changes',
  }
}
```

### 2.5 隔离级别选择

```typescript
// 决策逻辑示意
function selectIsolationLevel(task: Task): IsolationLevel {
  // 只读任务 → 不需要隔离
  if (task.isReadOnly) {
    return 'none'
  }
  
  // 写入操作 + 可能有冲突 → Worktree 隔离
  if (task.requiresWriteAccess && task.mayConflict) {
    return 'worktree'
  }
  
  // 高风险操作 → 远程沙箱
  if (task.isHighRisk) {
    return 'remote'
  }
  
  // 默认 → 进程内隔离
  return 'in-process'
}
```

---

## 3. 设计要点

### 3.1 最小隔离原则

不是所有任务都需要 Worktree：
- **只读任务**（Explore、搜索）：不需要隔离
- **小改动**（单文件编辑）：进程内隔离足够
- **大改动**（多文件重构）：Worktree 隔离
- **高风险**（生产部署）：远程沙箱

### 3.2 自动清理 vs 手动清理

| 场景 | 自动清理 | 手动清理 |
|------|---------|---------|
| 无改动 | ✅ | |
| 有改动但已提交 | ✅ | |
| 有未提交改动 | | ✅ 提示用户 |
| 任务失败 | | ✅ 保留供调试 |
| 用户取消 | | ✅ 提示用户 |

### 3.3 隔离与通信的平衡

Worktree 提供文件系统隔离，但 Agent 仍需要协作：

```
┌──────────────────┐     ┌──────────────────┐
│   主工作目录      │     │   Worktree       │
│  (Coordinator)   │     │   (Worker)       │
├──────────────────┤     ├──────────────────┤
│                  │     │                  │
│  Scratchpad ◄────┼─────┼───► Scratchpad   │
│  (共享知识)       │     │   (共享知识)      │
│                  │     │                  │
│  Git 远程仓库 ◄──┼─────┼───► Git 远程仓库  │
│  (最终同步)       │     │   (最终同步)      │
│                  │     │                  │
└──────────────────┘     └──────────────────┘
```

### 3.4 资源成本考虑

每个 Worktree 占用磁盘空间（工作区文件），但共享 .git 目录，所以：
- 小项目：Worktree 成本低
- 大项目（node_modules）：考虑清理策略或符号链接

---

## 4. 可落地方案

### 4.1 Worktree 管理框架

```typescript
// ===== 类型定义 =====
export interface WorktreeInfo {
  path: string
  branch: string
  parentPath: string
  createdAt: number
  taskId: string
}

export interface WorktreeManager {
  create(parentPath: string, taskId: string): Promise<WorktreeInfo>
  cleanup(path: string): Promise<CleanupResult>
  list(): Promise<WorktreeInfo[]>
}

// ===== 实现 =====
export class GitWorktreeManager implements WorktreeManager {
  private worktrees = new Map<string, WorktreeInfo>()

  async create(parentPath: string, taskId: string): Promise<WorktreeInfo> {
    // 1. 生成唯一分支名
    const branchName = `agent/${taskId}/${Date.now()}`
    
    // 2. 生成 Worktree 路径
    const worktreePath = join(
      dirname(parentPath),
      `${basename(parentPath)}-worktrees`,
      taskId,
    )
    
    // 3. 创建 Worktree
    await execFile('git', [
      'worktree', 'add',
      '-b', branchName,
      worktreePath,
    ], { cwd: parentPath })
    
    // 4. 记录信息
    const info: WorktreeInfo = {
      path: worktreePath,
      branch: branchName,
      parentPath,
      createdAt: Date.now(),
      taskId,
    }
    this.worktrees.set(worktreePath, info)
    
    return info
  }

  async cleanup(path: string): Promise<CleanupResult> {
    const info = this.worktrees.get(path)
    if (!info) {
      return { status: 'not_found' }
    }

    // 检查是否有改动
    const { stdout } = await execFile(
      'git',
      ['status', '--porcelain'],
      { cwd: path },
    )
    
    const hasChanges = stdout.trim().length > 0
    
    if (!hasChanges) {
      // 无改动，直接删除
      await this.deleteWorktree(path, info.branch)
      this.worktrees.delete(path)
      return { status: 'deleted' }
    }
    
    // 有改动，保留
    return {
      status: 'preserved',
      path,
      branch: info.branch,
      changes: stdout.trim().split('\n'),
    }
  }

  private async deleteWorktree(path: string, branch: string): Promise<void> {
    // 删除 Worktree
    await execFile('git', ['worktree', 'remove', path], { cwd: path })
    // 删除分支
    await execFile('git', ['branch', '-D', branch], { cwd: path })
  }

  async list(): Promise<WorktreeInfo[]> {
    return Array.from(this.worktrees.values())
  }
}
```

### 4.2 隔离策略选择器

```typescript
export type IsolationLevel = 'none' | 'in-process' | 'worktree' | 'remote'

export class IsolationSelector {
  constructor(private config: IsolationConfig) {}

  select(task: TaskDescriptor): IsolationLevel {
    // 只读任务不需要隔离
    if (task.isReadOnly) {
      return 'none'
    }

    // 检查是否需要并行执行
    const runningTasks = this.getRunningTasks()
    const potentialConflict = runningTasks.some(t => 
      this.overlaps(t.affectedFiles, task.affectedFiles)
    )

    if (potentialConflict) {
      // 可能冲突，使用 Worktree
      return 'worktree'
    }

    // 单 Agent 或无冲突
    return 'in-process'
  }

  private overlaps(files1: string[], files2: string[]): boolean {
    // 检查文件路径是否有重叠
    const dirs1 = new Set(files1.map(f => dirname(f)))
    const dirs2 = new Set(files2.map(f => dirname(f)))
    
    for (const d of dirs1) {
      if (dirs2.has(d)) return true
    }
    return false
  }
}
```

### 4.3 非 Git 项目的适配

如果项目不在 Git 仓库中，Worktree 不可用，可以降级为目录级隔离：

```typescript
export class DirectoryIsolation implements IsolationStrategy {
  async create(taskId: string): Promise<string> {
    // 创建副本目录
    const sourcePath = getCwd()
    const isolatedPath = join(
      dirname(sourcePath),
      `${basename(sourcePath)}-isolated-${taskId}`,
    )
    
    // 复制文件（排除 node_modules 等）
    await copyDirectory(sourcePath, isolatedPath, {
      exclude: ['node_modules', '.git', 'dist', 'build'],
    })
    
    return isolatedPath
  }

  async cleanup(path: string): Promise<CleanupResult> {
    // 比较变更
    const changes = await diffDirectories(getCwd(), path)
    
    if (changes.length === 0) {
      await rm(path, { recursive: true })
      return { status: 'deleted' }
    }
    
    return { status: 'preserved', path, changes }
  }
}
```

---

## 5. 总结

Claude Code 的 Worktree 隔离设计体现了**分级隔离 + 按需使用**的理念：

1. **四级隔离**：无隔离 → 进程级 → Worktree → 远程沙箱
2. **自动创建清理**：任务启动时按需创建，完成时自动清理
3. **保留有价值的改动**：未提交的改动保留 Worktree，供用户审查
4. **Scratchpad 共享**：隔离的工作区通过共享目录协作
5. **降级方案**：非 Git 项目使用目录副本隔离

这套设计在多 Agent 并行场景下有效避免了文件冲突，是文件系统级隔离的标准方案。
