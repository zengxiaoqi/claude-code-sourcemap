# 设计模式：Agent 记忆三级分离

## 1. 模式概述

**问题**：Agent 需要记忆来保持上下文和知识，但不同类型的知识有不同的作用域和生命周期。全局记忆会污染不同项目，项目记忆不共享会重复工作。

**解决方案**：三级分离的记忆作用域——user / project / local，每种作用域有独立的存储位置和生命周期。

**三级分离示意图**：

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent 记忆三级架构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Level 1: User Scope（用户级）                               │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  路径：~/.claude/agent-memory/<agentType>/            │ │
│  │  作用域：跨项目通用知识                                  │ │
│  │  生命周期：永久，跟随用户                                 │ │
│  │  内容示例：                                             │ │
│  │    - 编码风格偏好                                       │ │
│  │    - 常用命令别名                                       │ │
│  │    - 工具使用技巧                                       │ │
│  │  VCS：不入库                                           │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           ▼                                 │
│  Level 2: Project Scope（项目级）                            │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  路径：<project>/.claude/agent-memory/<agentType>/    │ │
│  │  作用域：项目特定知识                                    │ │
│  │  生命周期：跟随项目                                      │ │
│  │  内容示例：                                             │ │
│  │    - 项目架构决策                                       │ │
│  │    - API 密钥位置                                       │ │
│  │    - 团队编码规范                                       │ │
│  │  VCS：可入库，团队成员共享                               │ │
│  └───────────────────────────────────────────────────────┘ │
│                           │                                 │
│                           ▼                                 │
│  Level 3: Local Scope（本地级）                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  路径：<project>/.claude/agent-memory-local/<agentType>/ │
│  │  作用域：本机特定知识                                    │ │
│  │  生命周期：跟随本地环境                                  │ │
│  │  内容示例：                                             │ │
│  │    - 本地环境变量                                       │ │
│  │    - 个人调试配置                                       │ │
│  │    - 临时实验记录                                       │ │
│  │  VCS：不入库（在 .gitignore 中）                         │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 源码实现分析

### 2.1 记忆作用域定义

> 文件：`tools/AgentTool/agentMemory.ts`

```typescript
// 持久化 Agent 记忆作用域
// 'user' (~/.claude/agent-memory/) — 跨项目通用知识
// 'project' (.claude/agent-memory/) — 项目特定知识（可提交到 VCS）
// 'local' (.claude/agent-memory-local/) — 本机特定知识（不入 VCS）
export type AgentMemoryScope = 'user' | 'project' | 'local'
```

### 2.2 记忆目录获取

> 文件：`tools/AgentTool/agentMemory.ts`

```typescript
/**
 * 返回指定 Agent 类型和作用域的记忆目录
 * - 'user' scope: <memoryBase>/agent-memory/<agentType>/
 * - 'project' scope: <cwd>/.claude/agent-memory/<agentType>/
 * - 'local' scope: 见 getLocalAgentMemoryDir()
 */
export function getAgentMemoryDir(
  agentType: string,
  scope: AgentMemoryScope,
): string {
  const dirName = sanitizeAgentTypeForPath(agentType)
  switch (scope) {
    case 'project':
      return join(getCwd(), '.claude', 'agent-memory', dirName) + sep
    case 'local':
      return getLocalAgentMemoryDir(dirName)
    case 'user':
      return join(getMemoryBaseDir(), 'agent-memory', dirName) + sep
  }
}

/**
 * 返回本地 Agent 记忆目录，项目特定且不入 VCS
 * 当 CLAUDE_CODE_REMOTE_MEMORY_DIR 设置时，持久化到挂载点（带项目命名空间）
 * 否则使用 <cwd>/.claude/agent-memory-local/<agentType>/
 */
function getLocalAgentMemoryDir(dirName: string): string {
  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    return (
      join(
        process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR,
        'projects',
        sanitizePath(
          findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot(),
        ),
        'agent-memory-local',
        dirName,
      ) + sep
    )
  }
  return join(getCwd(), '.claude', 'agent-memory-local', dirName) + sep
}
```

### 2.3 记忆路径验证

> 文件：`tools/AgentTool/agentMemory.ts`

```typescript
// 检查文件是否在 Agent 记忆目录中（任意作用域）
export function isAgentMemoryPath(absolutePath: string): boolean {
  // 安全：规范化防止通过 .. 段绕过路径遍历
  const normalizedPath = normalize(absolutePath)
  const memoryBase = getMemoryBaseDir()

  // User 作用域：检查 memory base（可能是自定义目录或 config home）
  if (normalizedPath.startsWith(join(memoryBase, 'agent-memory') + sep)) {
    return true
  }

  // Project 作用域：始终基于 cwd（不会被重定向）
  if (
    normalizedPath.startsWith(join(getCwd(), '.claude', 'agent-memory') + sep)
  ) {
    return true
  }

  // Local 作用域：设置 REMOTE_MEMORY_DIR 时持久化到挂载点，
  // 否则基于 cwd
  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    if (
      normalizedPath.includes(sep + 'agent-memory-local' + sep) &&
      normalizedPath.startsWith(
        join(process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR, 'projects') + sep,
      )
    ) {
      return true
    }
  } else if (
    normalizedPath.startsWith(
      join(getCwd(), '.claude', 'agent-memory-local') + sep,
    )
  ) {
    return true
  }

  return false
}
```

### 2.4 Agent 类型名称清理

```typescript
/**
 * 清理 Agent 类型名称用于目录名
 * 将冒号（Windows 无效，用于插件命名空间的 Agent 类型如 "my-plugin:my-agent"）
 * 替换为连字符
 */
function sanitizeAgentTypeForPath(agentType: string): string {
  return agentType.replace(/:/g, '-')
}
```

### 2.5 记忆目录管理

> 文件：`memdir/memdir.ts`

```typescript
// 确保记忆目录存在
export function ensureMemoryDirExists(dir: string): void {
  if (!existsSync(dir)) {
    mkdirSync(dir, { recursive: true })
  }
}

// 构建记忆 Prompt
export function buildMemoryPrompt(dir: string): string {
  const files = readdirSync(dir).filter(f => f.endsWith('.md'))
  if (files.length === 0) return ''
  
  let prompt = `# Agent Memory\n\nThis directory contains persistent knowledge for this agent type.\n\n`
  
  for (const file of files) {
    const content = readFileSync(join(dir, file), 'utf-8')
    prompt += `## ${file}\n\n${content}\n\n`
  }
  
  return prompt
}
```

### 2.6 记忆在 Agent 中的使用

> 文件：`tools/AgentTool/runAgent.ts`

Agent 启动时会加载其记忆：

```typescript
// 加载 Agent 特定的 MCP 服务器
// Agent 可以定义自己的 MCP 服务器（附加到父级的 MCP 客户端）
async function initializeAgentMcpServers(
  agentDefinition: AgentDefinition,
  parentClients: MCPServerConnection[],
): Promise<{
  clients: MCPServerConnection[]
  tools: Tools
  cleanup: () => Promise<void>
}> {
  // ...
}

// Agent 记忆加载（伪代码）
async function loadAgentMemory(agentType: string): Promise<string> {
  const memories: string[] = []
  
  // 按优先级加载：user → project → local（后加载的覆盖前面的）
  for (const scope of ['user', 'project', 'local'] as const) {
    const dir = getAgentMemoryDir(agentType, scope)
    const prompt = buildMemoryPrompt(dir)
    if (prompt) memories.push(prompt)
  }
  
  return memories.join('\n\n---\n\n')
}
```

---

## 3. 设计要点

### 3.1 作用域隔离

三个作用域完全独立，不会互相污染：

```
User Scope                 Project Scope              Local Scope
    │                           │                          │
    ├─ 编码风格                  ├─ 项目架构                 ├─ 本地环境
    ├─ 工具偏好                  ├─ API 位置                 ├─ 调试配置
    └─ 通用知识                  └─ 团队规范                 └─ 临时记录
    │                           │                          │
    ▼                           ▼                          ▼
~/.claude/                .claude/                  .claude/
agent-memory/             agent-memory/             agent-memory-local/
    │                           │                          │
    └─ 不会意外提交              └─ 可提交共享               └─ gitignore
```

### 3.2 优先级与覆盖

当同一键在不同作用域存在时，后加载的覆盖前面的：

```
加载顺序：User → Project → Local
优先级：  低      中       高

例如：
  User:    "使用 2 空格缩进"
  Project: "本项目使用 4 空格缩进"  ← 覆盖 User
  Local:   "临时使用 tab 缩进"      ← 覆盖 Project（如果存在）
```

### 3.3 VCS 友好

| 作用域 | Git 状态 | 共享方式 |
|--------|---------|---------|
| User | 不入库 | 个人配置，不共享 |
| Project | 可入库 | 团队共享，提交到仓库 |
| Local | gitignore | 本机特定，不共享 |

### 3.4 远程记忆支持

`CLAUDE_CODE_REMOTE_MEMORY_DIR` 环境变量允许将 local 作用域持久化到远程挂载点：

```bash
export CLAUDE_CODE_REMOTE_MEMORY_DIR=/mnt/shared/claude-memory
```

这使得：
- 多台机器可以共享 local 记忆
- 云端备份 Agent 记忆
- 容器环境中持久化记忆

### 3.5 安全考虑

```typescript
// 路径规范化防止遍历攻击
const normalizedPath = normalize(absolutePath)

// 插件 Agent 类型的冒号清理
function sanitizeAgentTypeForPath(agentType: string): string {
  return agentType.replace(/:/g, '-')
}
```

---

## 4. 可落地方案

### 4.1 通用记忆框架

```typescript
// ===== 类型定义 =====
export type MemoryScope = 'user' | 'project' | 'local'

export interface MemoryEntry {
  key: string
  value: string
  scope: MemoryScope
  updatedAt: number
  tags?: string[]
}

export interface MemoryConfig {
  userBaseDir?: string   // 默认 ~/.claude/
  projectBaseDir?: string // 默认 <cwd>/.claude/
  remoteDir?: string      // CLAUDE_CODE_REMOTE_MEMORY_DIR
}

// ===== 记忆管理器 =====
export class AgentMemoryManager {
  private config: MemoryConfig

  constructor(config: MemoryConfig = {}) {
    this.config = config
  }

  // 获取记忆目录
  getMemoryDir(agentType: string, scope: MemoryScope): string {
    const dirName = this.sanitizeAgentType(agentType)
    
    switch (scope) {
      case 'user':
        return join(
          this.config.userBaseDir ?? join(homedir(), '.claude'),
          'agent-memory',
          dirName,
        )
      case 'project':
        return join(
          this.config.projectBaseDir ?? process.cwd(),
          '.claude',
          'agent-memory',
          dirName,
        )
      case 'local':
        if (this.config.remoteDir) {
          return join(
            this.config.remoteDir,
            'projects',
            this.getProjectHash(),
            'agent-memory-local',
            dirName,
          )
        }
        return join(
          this.config.projectBaseDir ?? process.cwd(),
          '.claude',
          'agent-memory-local',
          dirName,
        )
    }
  }

  // 读取记忆
  async read(
    agentType: string,
    key: string,
    scope?: MemoryScope,
  ): Promise<string | null> {
    const scopes: MemoryScope[] = scope 
      ? [scope] 
      : ['user', 'project', 'local'] // 按优先级尝试

    for (const s of scopes) {
      const dir = this.getMemoryDir(agentType, s)
      const filePath = join(dir, `${key}.md`)
      try {
        return await fs.readFile(filePath, 'utf-8')
      } catch {
        continue
      }
    }
    return null
  }

  // 写入记忆
  async write(
    agentType: string,
    key: string,
    value: string,
    scope: MemoryScope = 'project',
  ): Promise<void> {
    const dir = this.getMemoryDir(agentType, scope)
    await fs.mkdir(dir, { recursive: true })
    const filePath = join(dir, `${key}.md`)
    await fs.writeFile(filePath, value, 'utf-8')
  }

  // 列出所有记忆
  async list(
    agentType: string,
    scope?: MemoryScope,
  ): Promise<MemoryEntry[]> {
    const scopes: MemoryScope[] = scope 
      ? [scope] 
      : ['user', 'project', 'local']
    
    const entries: MemoryEntry[] = []
    
    for (const s of scopes) {
      const dir = this.getMemoryDir(agentType, s)
      try {
        const files = await fs.readdir(dir)
        for (const file of files) {
          if (file.endsWith('.md')) {
            const stat = await fs.stat(join(dir, file))
            entries.push({
              key: file.slice(0, -3),
              value: await fs.readFile(join(dir, file), 'utf-8'),
              scope: s,
              updatedAt: stat.mtimeMs,
            })
          }
        }
      } catch {
        continue
      }
    }
    
    return entries
  }

  // 构建 Prompt
  buildPrompt(agentType: string): string {
    const entries = await this.list(agentType)
    if (entries.length === 0) return ''

    const grouped = this.groupByScope(entries)
    let prompt = '# Agent Memory\n\n'

    for (const [scope, items] of Object.entries(grouped)) {
      prompt += `## ${scope.toUpperCase()} Scope\n\n`
      for (const item of items) {
        prompt += `### ${item.key}\n\n${item.value}\n\n`
      }
    }

    return prompt
  }

  // 辅助方法
  private sanitizeAgentType(type: string): string {
    return type.replace(/:/g, '-')
  }

  private getProjectHash(): string {
    // 基于 Git 根目录或 cwd 生成唯一标识
    return createHash('sha256')
      .update(process.cwd())
      .digest('hex')
      .slice(0, 16)
  }

  private groupByScope(
    entries: MemoryEntry[],
  ): Record<MemoryScope, MemoryEntry[]> {
    return entries.reduce((acc, e) => {
      if (!acc[e.scope]) acc[e.scope] = []
      acc[e.scope].push(e)
      return acc
    }, {} as Record<MemoryScope, MemoryEntry[]>)
  }
}
```

### 4.2 集成到 Agent 系统

```typescript
export class Agent {
  private memory: AgentMemoryManager

  constructor(
    private definition: AgentDefinition,
    memoryConfig?: MemoryConfig,
  ) {
    this.memory = new AgentMemoryManager(memoryConfig)
  }

  async initialize(): Promise<void> {
    // 加载记忆到上下文
    const memoryPrompt = this.memory.buildPrompt(this.definition.agentType)
    this.context.addSystemPrompt(memoryPrompt)
  }

  async learn(key: string, value: string): Promise<void> {
    // Agent 学习新知识
    await this.memory.write(
      this.definition.agentType,
      key,
      value,
      'project', // 默认写入 project 作用域
    )
  }
}
```

### 4.3 高级场景

**团队共享记忆**：

```typescript
// .claude/agent-memory/general-purpose/project-conventions.md
// 提交到 Git，团队成员共享

# 项目约定

## 分支命名
- feature/* - 新功能
- fix/* - Bug 修复
- refactor/* - 重构

## 提交格式
遵循 Conventional Commits 规范
```

**隐私保护**：

```typescript
// 敏感信息只存 local 作用域
await memory.write(
  'general-purpose',
  'api-keys',
  'Stripe: sk_test_xxx',
  'local',  // 不入库
)
```

**记忆版本管理**：

```typescript
// project 作用域的记忆可以像代码一样版本化
git add .claude/agent-memory/
git commit -m "Update agent memory with new API patterns"
```

---

## 5. 总结

Claude Code 的 Agent 记忆三级分离设计体现了**作用域隔离 + VCS 友好**的理念：

1. **三个作用域**：user（跨项目）、project（团队共享）、local（本机特定）
2. **优先级覆盖**：local > project > user
3. **VCS 集成**：project 可入库，user/local 自动排除
4. **远程支持**：`CLAUDE_CODE_REMOTE_MEMORY_DIR` 实现云端持久化
5. **安全防护**：路径规范化 + Agent 类型名称清理

这套设计让 Agent 可以在不同粒度上积累知识，既支持个人偏好，又支持团队协作，是多 Agent 系统记忆管理的标准方案。
