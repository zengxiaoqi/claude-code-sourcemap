# Claude Code Memory 记忆系统设计深度分析

## 1. 架构总览

Claude Code 的 Memory 系统是一套**基于文件的持久化知识管理框架**，让 AI 跨会话积累知识。它不是一个简单的存储层，而是一个包含**写入、提取、整合、压缩、搜索**的完整生命周期系统。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Memory 系统架构全景                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │  写入路径 ①   │     │  写入路径 ②   │     │  写入路径 ③   │        │
│  │ 主Agent直接写 │     │ 后台提取Agent │     │ Dream整合    │        │
│  │ (Write/Edit) │     │ (extractMem) │     │ (consolidate)│        │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘        │
│         │                    │                    │                 │
│         └────────────────────┼────────────────────┘                 │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    存储层（文件系统）                          │  │
│  │                                                              │  │
│  │  ~/.claude/projects/<slug>/memory/                           │  │
│  │  ├── MEMORY.md          ← 索引入口（200行/25KB上限）          │  │
│  │  ├── user_role.md       ← 用户记忆                          │  │
│  │  ├── feedback_testing.md ← 反馈记忆                         │  │
│  │  ├── project_auth.md    ← 项目记忆                          │  │
│  │  ├── reference_links.md ← 引用记忆                          │  │
│  │  ├── logs/YYYY/MM/YYYY-MM-DD.md ← 日志（KAIROS模式）        │  │
│  │  └── team/              ← 团队记忆（TEAMMEM模式）            │  │
│  │                                                              │  │
│  │  ~/.claude/agent-memory/<agentType>/  ← Agent 级记忆         │  │
│  │                                                              │  │
│  │  .claude/agent-memory/<agentType>/    ← 项目级 Agent 记忆    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    读取层                                     │  │
│  │  ├── loadMemoryPrompt() → 系统提示注入                       │  │
│  │  ├── buildMemoryPrompt() → Agent 记忆注入                    │  │
│  │  ├── scanMemoryFiles() → 记忆扫描                            │  │
│  │  ├── findRelevantMemories() → 相关性检索                     │  │
│  │  └── grep/grep 过去会话 → 深度搜索                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 记忆类型分类

> 文件：`memdir/memoryTypes.ts`

Claude Code 将记忆严格限定为**四种类型**，这是一个经过评估验证的封闭分类法：

| 类型 | 用途 | 保存时机 | 使用时机 |
|------|------|---------|---------|
| **user** | 用户角色、偏好、知识水平 | 了解用户信息时 | 个性化交互方式 |
| **feedback** | 用户给出的指导（纠正+确认） | 被纠正或被确认时 | 避免重复犯同样错误 |
| **project** | 项目上下文（非代码可推导的） | 了解项目动态时 | 理解请求背景 |
| **reference** | 外部系统资源指针 | 发现外部资源时 | 定位外部信息 |

**关键设计原则——不要保存什么**：
- 代码模式/架构 → 可通过读代码推导
- Git 历史 → `git log/blame` 是权威来源
- 调试方案 → 修复已在代码中
- CLAUDE.md 中已有的内容 → 避免重复
- 临时任务状态 → 不需要跨会话持久化

```typescript
// 明确的不保存清单
export const WHAT_NOT_TO_SAVE_SECTION = [
  '## What NOT to save in memory',
  '- Code patterns, conventions, architecture, file paths, or project structure',
  '- Git history, recent changes, or who-changed-what',
  '- Debugging solutions or fix recipes',
  '- Anything already documented in CLAUDE.md files.',
  '- Ephemeral task details: in-progress work, temporary state, current conversation context.',
  // 即使用户要求保存也要拒绝
  'These exclusions apply even when the user explicitly asks you to save.',
]
```

### 2.1 记忆文件格式

每个记忆文件使用 **frontmatter + markdown** 格式：

```markdown
---
name: 用户偏好-简洁输出
description: 用户希望简洁的回复，不要在末尾总结
type: feedback
---

## 规则
不要在每个响应末尾总结已做的事情，用户可以自己读 diff。

**Why:** 用户明确表示不需要尾随总结
**How to apply:** 完成操作后直接结束，不加总结段落
```

### 2.2 索引文件 MEMORY.md

`MEMORY.md` 是所有记忆的**索引**，不是记忆本身：

```markdown
- [用户角色](user_role.md) — 数据科学家，关注可观测性
- [测试反馈](feedback_testing.md) — 集成测试必须使用真实数据库
- [认证项目](project_auth.md) — 认证中间件重写因合规需求
- [监控引用](reference_grafana.md) — grafana.internal/d/api-latency 延迟面板
```

**严格限制**：200 行 / 25KB 上限，超出自动截断并警告。

---

## 3. 存储路径体系

> 文件：`memdir/paths.ts`

### 3.1 自动记忆路径

```
默认路径：~/.claude/projects/<sanitized-git-root>/memory/
                    │
                    ├── MEMORY.md              ← 索引
                    ├── user_role.md           ← 记忆文件
                    └── ...

覆盖优先级（从高到低）：
1. CLAUDE_COWORK_MEMORY_PATH_OVERRIDE 环境变量（完整路径覆盖）
2. settings.json 中的 autoMemoryDirectory（支持 ~/ 展开）
3. 默认计算路径
```

**路径计算使用 Git 根目录**（非 CWD），确保同一仓库的不同 Worktree 共享记忆：

```typescript
function getAutoMemBase(): string {
  return findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()
}
```

### 3.2 Agent 记忆路径

> 文件：`tools/AgentTool/agentMemory.ts`

三级分离（已在 `06-agent-memory-scopes.md` 详细分析）：

| 作用域 | 路径 | VCS |
|--------|------|-----|
| user | `~/.claude/agent-memory/<agentType>/` | 不入库 |
| project | `.claude/agent-memory/<agentType>/` | 可入库 |
| local | `.claude/agent-memory-local/<agentType>/` | gitignore |

### 3.3 安全验证

```typescript
// 路径安全校验：拒绝危险路径
function validateMemoryPath(raw: string | undefined, expandTilde: boolean): string | undefined {
  // 拒绝：相对路径、根目录、Windows 驱动器根、UNC 路径、null 字节
  if (!isAbsolute(normalized) || normalized.length < 3 || 
      /^[A-Za-z]:$/.test(normalized) || normalized.startsWith('\\\\') ||
      normalized.includes('\0')) {
    return undefined
  }
  return normalized + sep
}
```

---

## 4. 三条写入路径

### 4.1 路径①：主 Agent 直接写入

主 Agent 在对话过程中直接使用 Write/Edit 工具写入记忆文件。系统提示中包含完整的记忆保存指南，模型根据对话内容自主判断何时保存。

**触发时机**：
- 用户说"记住这个"
- 模型学到重要的用户偏好
- 模型发现重要的项目上下文

### 4.2 路径②：后台提取 Agent（extractMemories）

> 文件：`services/extractMemories/extractMemories.ts`

**核心设计**：使用 **forked agent** 模式——完美复制主对话，共享 Prompt Cache。

```
用户消息 → 主Agent处理 → 模型回复完成
                              │
                              ▼
                    handleStopHooks 触发
                              │
                              ▼
                    extractMemories（异步，不阻塞）
                              │
                              ├── Fork 主对话（共享 Prompt Cache）
                              ├── 注入提取 Prompt
                              ├── 后台 Agent 扫描对话，提取可持久化的知识
                              ├── 写入记忆文件
                              └── 更新 MEMORY.md 索引
```

**关键机制**：

```typescript
// 1. 互斥设计：主Agent写了记忆 → 后台Agent跳过
function hasMemoryWritesSince(messages: Message[], sinceUuid: string): boolean {
  // 检查主Agent是否已经写了记忆文件
  // 如果写了，后台Agent跳过该范围，避免重复
}

// 2. 光标追踪：只处理新增消息
let lastMemoryMessageUuid: string | undefined
// 每次提取后更新光标，下次只处理光标之后的消息

// 3. 合并执行：多次请求合并为一次
if (inProgress) {
  pendingContext = { context, appendSystemMessage }  // 暂存最新上下文
  return  // 等当前提取完成后自动跑 trailing run
}

// 4. 限流保护：5轮上限
maxTurns: 5  // 后台Agent最多5轮，防止陷入验证循环
```

**提取 Prompt 结构**：
- 注入当前记忆清单（`scanMemoryFiles`）→ 避免重复创建
- 只给 Grep/Read/Write/Edit 工具 → 最小权限
- 明确指导：合并到已有文件 > 创建新文件

### 4.3 路径③：Dream 整合（consolidation）

> 文件：`services/autoDream/consolidationPrompt.ts`

Dream 是一种**反思性记忆整合**，定期或在空闲时运行，将原始记忆提炼为结构化知识。

```
Dream 整合流程（4个阶段）：

Phase 1 — 定向（Orient）
  ├── ls 记忆目录，了解现有文件
  ├── 读取 MEMORY.md 索引
  └── 浏览已有主题文件

Phase 2 — 收集信号（Gather）
  ├── 优先：Daily logs（KAIROS 模式的追加日志）
  ├── 其次：已有但过时的记忆
  └── 最后：会话 Transcript（grep，不全文读取）

Phase 3 — 整合（Consolidate）
  ├── 合并新信息到已有文件（而非创建重复文件）
  ├── 转换相对日期为绝对日期
  └── 删除被推翻的事实

Phase 4 — 修剪和索引（Prune & Index）
  ├── 更新 MEMORY.md 索引（≤200行/≤25KB）
  ├── 删除过时/错误的指针
  ├── 精简过长的索引行（≤150字符）
  └── 解决矛盾：两个文件冲突时修正错误的那个
```

---

## 5. 读取与检索

### 5.1 系统提示注入

> 文件：`memdir/memdir.ts` — `loadMemoryPrompt()`

记忆在会话启动时通过**系统提示**注入：

```typescript
async function loadMemoryPrompt(): Promise<string | null> {
  // 分支逻辑：
  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    // KAIROS 模式：使用 Daily Log Prompt
    return buildAssistantDailyLogPrompt(skipIndex)
  }

  if (feature('TEAMMEM') && isTeamMemoryEnabled()) {
    // 团队记忆：合并 Prompt（私人 + 团队）
    return teamMemPrompts.buildCombinedMemoryPrompt(...)
  }

  if (autoEnabled) {
    // 个人记忆：标准 Prompt
    return buildMemoryLines('auto memory', autoDir, ...).join('\n')
  }

  return null  // 记忆禁用
}
```

### 5.2 记忆扫描

> 文件：`memdir/memoryScan.ts`

```typescript
// 扫描记忆文件，提取 frontmatter 元数据
async function scanMemoryFiles(memoryDir: string, signal: AbortSignal): Promise<MemoryHeader[]> {
  const entries = await readdir(memoryDir, { recursive: true })
  const mdFiles = entries.filter(f => f.endsWith('.md') && basename(f) !== 'MEMORY.md')

  // 并行读取每个文件的 frontmatter（最多30行）
  const headers = await Promise.allSettled(
    mdFiles.map(async f => {
      const { content, mtimeMs } = await readFileInRange(filePath, 0, 30, ...)
      const { frontmatter } = parseFrontmatter(content)
      return { filename, filePath, mtimeMs, description, type }
    })
  )

  // 按修改时间降序排列，最多200个文件
  return headers.sort((a, b) => b.mtimeMs - a.mtimeMs).slice(0, 200)
}
```

### 5.3 相关性检索

> 文件：`memdir/findRelevantMemories.ts`

系统不仅加载 MEMORY.md 索引，还能根据当前上下文搜索相关记忆：

```
检索策略（从快到慢）：
1. MEMORY.md 索引（始终加载，≤200行）
2. 记忆文件 frontmatter 扫描（description 匹配）
3. Grep 记忆目录内容（按需）
4. Grep 过去会话 Transcript（最后手段，JSONL 文件很大）
```

### 5.4 信任与验证

> 文件：`memdir/memoryTypes.ts` — `TRUSTING_RECALL_SECTION`

记忆可能过时，系统要求模型在引用前**验证**：

```
## Before recommending from memory

A memory that names a specific function, file, or flag is a claim 
that it existed *when the memory was written*. It may have been 
renamed, removed, or never merged. Before recommending it:

- If the memory names a file path: check the file exists.
- If the memory names a function or flag: grep for it.
- If the user is about to act on your recommendation, verify first.

"The memory says X exists" is not the same as "X exists now."
```

---

## 6. 特殊模式

### 6.1 KAIROS 助手模式（Daily Log）

> 文件：`memdir/memdir.ts` — `buildAssistantDailyLogPrompt()`

长生命周期的助手会话使用**追加日志**而非维护 MEMORY.md：

```
记忆目录/memory/
  ├── MEMORY.md          ← 夜间从日志蒸馏生成
  ├── user_role.md       ← 蒸馏后的主题文件
  └── logs/
      └── 2026/
          └── 04/
              ├── 2026-04-04.md   ← 追加日志
              └── 2026-04-05.md   ← 追加日志
```

**设计理由**：
- 助手会话是持久化的，不适合频繁修改 MEMORY.md
- 追加日志是**只写**操作，天然无冲突
- 夜间 Dream 进程将日志蒸馏为主题文件 + MEMORY.md

### 6.2 团队记忆（TEAMMEM）

> 文件：`memdir/teamMemPaths.ts` + `memdir/teamMemPrompts.ts`

团队记忆是自动记忆目录的**子目录**：

```
~/.claude/projects/<slug>/memory/     ← 个人记忆
~/.claude/projects/<slug>/memory/team/ ← 团队记忆
```

**记忆类型与作用域映射**：

| 类型 | 默认作用域 | 说明 |
|------|-----------|------|
| user | **私人** | 始终私人，不共享 |
| feedback | 默认私人 | 项目级约定可存团队 |
| project | 偏向团队 | 大多数项目上下文对团队有价值 |
| reference | **团队** | 外部资源指针对所有人有用 |

---

## 7. 启用控制

> 文件：`memdir/paths.ts` — `isAutoMemoryEnabled()`

```typescript
// 优先级链（第一个定义的生效）：
function isAutoMemoryEnabled(): boolean {
  // 1. 环境变量（最高优先级）
  if (isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_AUTO_MEMORY)) return false
  if (isEnvDefinedFalsy(envVal)) return true

  // 2. --bare 模式 → 关闭
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) return false

  // 3. 远程模式无持久存储 → 关闭
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) && !CLAUDE_CODE_REMOTE_MEMORY_DIR) return false

  // 4. settings.json 配置
  if (settings.autoMemoryEnabled !== undefined) return settings.autoMemoryEnabled

  // 5. 默认：启用
  return true
}
```

---

## 8. Session Memory（会话记忆）

> 文件：`services/SessionMemory/sessionMemory.ts`

除了持久化记忆，还有**会话级记忆**——维护当前会话的笔记文件：

```
工作原理：
1. 周期性后台运行 forked agent
2. 从当前对话提取关键信息
3. 写入会话记忆文件（非持久化）
4. 用于上下文压缩后恢复关键信息

阈值控制：
- 初始化阈值：达到一定对话量后开始
- 更新阈值：每次更新之间需要足够的工具调用次数
```

---

## 9. 设计模式总结

| 模式 | 应用 | 价值 |
|------|------|------|
| **文件即记忆** | 所有记忆存储为 .md 文件 | 透明、可编辑、可版本化 |
| **索引 + 详情分离** | MEMORY.md 索引 + 主题文件 | 索引小（≤200行），详情无限 |
| **Frontmatter 元数据** | name/description/type | 结构化查询，不依赖文件名 |
| **三条写入路径** | 主Agent/后台提取/Dream | 覆盖即时、遗漏、整合三种场景 |
| **Forked Agent 提取** | 共享 Prompt Cache 的后台 Agent | 零额外成本提取记忆 |
| **光标追踪** | lastMemoryMessageUuid | 只处理增量，避免重复 |
| **互斥设计** | hasMemoryWritesSince | 主Agent和后台Agent不同时写 |
| **追加日志** | KAIROS Daily Log | 无冲突写入，适合长会话 |
| **信任验证** | TRUSTING_RECALL_SECTION | 记忆可能过时，使用前验证 |
| **封闭分类法** | 4种记忆类型 | 明确边界，避免随意保存 |

---

## 10. 可借鉴要点

1. **索引 + 详情分离**：MEMORY.md 作为轻量索引，详情存储在独立文件中。索引始终加载（≤200行），详情按需读取。

2. **三条写入路径互补**：
   - 主Agent即时写（用户说"记住"）
   - 后台Agent补漏写（遗漏的重要信息）
   - Dream整合写（定期提炼整理）

3. **Forked Agent 提取**：后台提取使用完美 Fork 的主对话，共享 Prompt Cache，额外成本极低。

4. **封闭分类法**：4种明确的记忆类型 + 明确的"不保存"清单，防止记忆膨胀。

5. **信任验证**：记忆是快照，不是真理。使用前必须验证。

6. **追加日志模式**：长生命周期场景用追加日志替代频繁修改，避免冲突。

7. **路径安全验证**：严格的路径校验防止目录遍历和权限提升攻击。
