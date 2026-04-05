# Commands 模块深度分析

## 模块概览

| 指标 | 值 |
|------|-----|
| 文件数 | ~189 |
| 核心职责 | 定义所有斜杠命令（`/` command），提供用户交互式操作入口，管理与工具（Tool）的区别与协作 |
| 关键文件 | `src/commands.ts`, `src/types/command.ts`, `src/commands/` 目录 |

### 关键文件列表

| 文件路径 | 作用 |
|----------|------|
| `src/commands.ts` | 命令注册中心，`getCommands()` 汇总所有命令源 |
| `src/types/command.ts` | 命令类型定义（`Command`, `PromptCommand`, `LocalCommand`, `LocalJSXCommand`） |
| `src/commands/help/index.ts` | `/help` 命令 |
| `src/commands/init.ts` | `/init` 命令（CLAUDE.md 生成） |
| `src/commands/compact/index.ts` | `/compact` 命令（上下文压缩） |
| `src/commands/config/index.ts` | `/config` 配置管理 |
| `src/commands/session/index.ts` | `/session` 会话管理 |
| `src/commands/skills/index.ts` | `/skills` 技能管理 |
| `src/commands/mcp/index.ts` | `/mcp` MCP 服务器管理 |
| `src/skills/loadSkillsDir.ts` | 技能目录加载 |
| `src/utils/plugins/loadPluginCommands.ts` | 插件命令加载 |

---

## 1. 完整命令清单

### 1.1 核心命令（始终可用）

| 命令 | 文件 | 功能 | 类型 |
|------|------|------|------|
| `/help` | `commands/help/` | 显示帮助和可用命令 | local-jsx |
| `/init` | `commands/init.ts` | 生成 CLAUDE.md | prompt |
| `/compact` | `commands/compact/` | 压缩对话上下文 | local-jsx |
| `/config` | `commands/config/` | 查看/修改配置 | local-jsx |
| `/clear` | `commands/clear/` | 清除对话 | local-jsx |
| `/cost` | `commands/cost/` | 显示 token 使用统计 | local-jsx |
| `/diff` | `commands/diff/` | 显示未提交的更改 | local-jsx |
| `/doctor` | `commands/doctor/` | 诊断安装问题 | local-jsx |
| `/memory` | `commands/memory/` | 管理 CLAUDE.md 记忆文件 | local-jsx |
| `/model` | `commands/model/` | 切换模型 | local-jsx |
| `/status` | `commands/status/` | 显示会话状态 | local-jsx |
| `/resume` | `commands/resume/` | 恢复之前的会话 | local-jsx |
| `/session` | `commands/session/` | 会话管理 | local-jsx |
| `/review` | `commands/review.ts` | 代码审查 | prompt |
| `/permissions` | `commands/permissions/` | 管理权限规则 | local-jsx |
| `/hooks` | `commands/hooks/` | 管理钩子 | local-jsx |
| `/plan` | `commands/plan/` | 计划模式 | local-jsx |
| `/files` | `commands/files/` | 显示文件上下文 | local-jsx |
| `/login` | `commands/login/` | 登录 | local-jsx |
| `/logout` | `commands/logout/` | 登出 | local-jsx |
| `/theme` | `commands/theme/` | 切换主题 | local-jsx |
| `/vim` | `commands/vim/` | Vim 模式切换 | local-jsx |
| `/bughunter` | `commands/bughunter/` | Bug 搜索 | prompt |
| `/security-review` | `commands/security-review.ts` | 安全审查 | prompt |
| `/upgrade` | `commands/upgrade/` | 升级 CLI | local-jsx |
| `/usage` | `commands/usage/` | 使用统计 | local-jsx |
| `/insights` | `commands/insights.ts` | 会话分析报告 | prompt |

### 1.2 配置与环境命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/add-dir` | `commands/add-dir/` | 添加工作目录 |
| `/mcp` | `commands/mcp/` | MCP 服务器管理 |
| `/ide` | `commands/ide/` | IDE 集成管理 |
| `/env` | `commands/env/` | 环境变量管理 |
| `/remote-env` | `commands/remote-env/` | 远程环境管理 |
| `/desktop` | `commands/desktop/` | 桌面集成 |
| `/mobile` | `commands/mobile/` | 移动端集成 |
| `/effort` | `commands/effort/` | 调整推理努力级别 |
| `/fast` | `commands/fast/` | 快速模式 |
| `/output-style` | `commands/output-style/` | 输出风格 |
| `/model` | `commands/model/` | 模型选择 |
| `/tag` | `commands/tag/` | 标签管理 |
| `/keybindings` | `commands/keybindings/` | 快捷键管理 |
| `/color` | `commands/color/` | 颜色设置 |
| `/sandbox-toggle` | `commands/sandbox-toggle/` | 沙箱切换 |
| `/terminal-setup` | `commands/terminalSetup/` | 终端设置 |
| `/privacy-settings` | `commands/privacy-settings/` | 隐私设置 |
| `/rate-limit-options` | `commands/rate-limit-options/` | 速率限制选项 |
| `/stats` | `commands/stats/` | 统计信息 |

### 1.3 会话与协作命令

| 命令 | 文件 | 功能 |
|------|------|------|
| `/rename` | `commands/rename/` | 重命名会话 |
| `/copy` | `commands/copy/` | 复制对话内容 |
| `/share` | `commands/share/` | 分享会话 |
| `/export` | `commands/export/` | 导出会话 |
| `/agents` | `commands/agents/` | 代理管理 |
| `/tasks` | `commands/tasks/` | 任务管理 |
| `/skills` | `commands/skills/` | 技能管理 |
| `/plugin` | `commands/plugin/` | 插件管理 |
| `/reload-plugins` | `commands/reload-plugins/` | 重载插件 |
| `/branch` | `commands/branch/` | Git 分支操作 |
| `/commit` | `commands/commit.ts` | Git 提交 |
| `/commit-push-pr` | `commands/commit-push-pr.ts` | 提交+推送+PR |
| `/feedback` | `commands/feedback/` | 提交反馈 |
| `/passes` | `commands/passes/` | 通行证管理 |

### 1.4 条件启用命令（Feature Flag 控制）

| 命令 | 条件 | 功能 |
|------|------|------|
| `/proactive` | `PROACTIVE` 或 `KAIROS` | 主动模式 |
| `/brief` | `KAIROS` | 简报模式 |
| `/assistant` | `KAIROS` | 助手模式 |
| `/bridge` | `BRIDGE_MODE` | 桥接模式 |
| `/remote-control-server` | `DAEMON && BRIDGE_MODE` | 远程控制服务器 |
| `/voice` | `VOICE_MODE` | 语音模式 |
| `/fork` | `FORK_SUBAGENT` | 分叉子代理 |
| `/buddy` | `BUDDY` | Buddy 模式 |
| `/peers` | `UDS_INBOX` | 对等节点 |
| `/workflows` | `WORKFLOW_SCRIPTS` | 工作流脚本 |
| `/torch` | `TORCH` | Torch 模式 |
| `/ultraplan` | `ULTRAPLAN` | 超级计划 |
| `/subscribe-pr` | `KAIROS_GITHUB_WEBHOOKS` | PR webhook 订阅 |
| `/force-snip` | `HISTORY_SNIP` | 强制裁剪 |
| `/agents-platform` | `USER_TYPE === 'ant'` | 平台代理 |
| `/autofix-pr` | `USER_TYPE === 'ant'` | 自动修复 PR |

### 1.5 动态命令源

除了内置命令外，还有四个动态命令源：

| 来源 | 加载函数 | 说明 |
|------|----------|------|
| 技能目录 | `getSkillDirCommands(cwd)` | 从 `.claude/skills/` 加载 |
| 插件技能 | `getPluginSkills()` | 从已安装插件加载 |
| 捆绑技能 | `getBundledSkills()` | 内置捆绑技能 |
| 内置插件技能 | `getBuiltinPluginSkillCommands()` | 内置插件的技能 |
| MCP prompts | MCP 协议的 prompts 功能 | MCP 服务器提供的命令 |

---

## 2. 命令注册与分发机制

### 2.1 命令类型系统

命令有三种核心类型（`src/types/command.ts`）：

```typescript
// 类型 1: prompt 命令 — 生成 prompt 文本发送给 LLM
type PromptCommand = {
  type: 'prompt'
  name: string
  description: string
  progressMessage: string
  contentLength: number
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
  // 可选属性
  allowedTools?: string[]
  model?: string
  context?: 'inline' | 'fork'
  agent?: string
  effort?: EffortValue
  paths?: string[]
}

// 类型 2: local 命令 — 执行本地逻辑，返回文本结果
type LocalCommand = {
  type: 'local'
  name: string
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}

// 类型 3: local-jsx 命令 — 执行本地逻辑，渲染 React UI
type LocalJSXCommand = {
  type: 'local-jsx'
  name: string
  load: () => Promise<LocalJSXCommandModule>
}
```

统一接口：

```typescript
type Command = (PromptCommand | LocalCommand | LocalJSXCommand) & CommandBase

type CommandBase = {
  name: string
  description: string
  aliases?: string[]
  isEnabled?: () => boolean
  isHidden?: boolean
  availability?: CommandAvailability[]
  argumentHint?: string
  whenToUse?: string
  version?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  immediate?: boolean
  isSensitive?: boolean
}
```

### 2.2 命令注册流程

**文件**：`src/commands.ts`

```typescript
// 1. 静态导入所有内置命令
import help from './commands/help/index.js'
import init from './commands/init.js'
import compact from './commands/compact/index.js'
// ...

// 2. 条件导入 feature-flag 控制的命令
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// 3. 命令列表（memoized）
const COMMANDS = memoize((): Command[] => [
  help, init, compact, config, clear, /* ... */
  ...(voiceCommand ? [voiceCommand] : []),
  ...(process.env.USER_TYPE === 'ant' ? INTERNAL_ONLY_COMMANDS : []),
])

// 4. 动态加载技能/插件命令
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [base, skills] = await Promise.all([
    Promise.resolve(COMMANDS()),
    getSkills(cwd),
  ])
  return [...base, ...skills.skillDirCommands, ...skills.pluginSkills, ...]
})
```

### 2.3 命令分发

```
用户输入 "/compact --full"
    ↓
1. 匹配命令名 → 查找 Command 对象
    ↓
2. 检查 isEnabled() 和 meetsAvailabilityRequirement()
    ↓
3. 根据 type 分发:
    ├── 'prompt'     → getPromptForCommand(args, context) → 注入对话
    ├── 'local'      → load().call(args, context)         → 显示文本结果
    └── 'local-jsx'  → load().call(onDone, context, args) → 渲染 React UI
```

### 2.4 可用性过滤

```typescript
// src/commands.ts
function meetsAvailabilityRequirement(cmd: Command): boolean {
  if (!cmd.availability) return true
  for (const a of cmd.availability) {
    switch (a) {
      case 'claude-ai': return isClaudeAISubscriber()
      case 'console': return !isClaudeAISubscriber() && !isUsing3PServices() && isFirstPartyAnthropicBaseUrl()
    }
  }
  return false
}
```

---

## 3. 命令与工具的区别和协作

### 3.1 核心区别

| 维度 | Command（斜杠命令） | Tool（工具） |
|------|---------------------|-------------|
| 调用者 | 用户主动输入 `/xxx` | LLM 决定调用 |
| 可见性 | 用户可见，在帮助中列出 | LLM 可见，通过 schema 描述 |
| 执行位置 | REPL 本地执行 | 工具执行管道 |
| 输出 | UI 渲染或 prompt 注入 | `tool_result` 返回 API |
| 目的 | 用户操作入口（设置、导航、管理） | LLM 能力扩展（文件、搜索、代码执行） |
| 权限 | 命令级过滤 | 多层权限检查 |

### 3.2 协作关系

1. **命令触发工具执行**：如 `/init` → LLM 使用 `Write` 工具写文件
2. **命令管理工具配置**：如 `/mcp` → 添加/删除 MCP 服务器 → 影响可用工具列表
3. **命令注入 prompt**：`PromptCommand` 生成的文本作为 LLM 对话的一部分，引导 LLM 使用特定工具
4. **技能 = 命令 + prompt**：`.claude/skills/` 下的技能文件注册为 `PromptCommand`，`/skill-name` 触发后注入预定义的 prompt

### 3.3 转化桥梁

```
Skill 目录 (.claude/skills/xxx/skill.md)
    ↓ getSkillDirCommands()
    ↓ 转换为 PromptCommand
    ↓ type: 'prompt', source: 'skills'
命令列表中的 /xxx 命令
    ↓ 用户调用
    ↓ getPromptForCommand()
    ↓ 读取 skill.md 内容
    ↓ 作为 ContentBlockParam 注入对话
LLM 理解 prompt → 选择合适的 Tool 执行
```

---

## 4. 命令的参数解析

### 4.1 非 Commander.js 集成

Commands 模块**没有使用 Commander.js**。参数解析采用以下策略：

1. **简单字符串参数**：命令的 `args` 参数是原始字符串，由各命令自行解析
2. **Prompt 命令**：`getPromptForCommand(args, context)` 接收完整参数字符串
3. **Local/LocalJSX 命令**：`call(args, context)` 或 `call(onDone, context, args)` 接收原始参数

示例（`/init` 命令）：

```typescript
// src/commands/init.ts
// init 是 PromptCommand 类型
// args 为空或包含子命令（如 --force）
// getPromptForCommand 返回 CLAUDE.md 生成 prompt
```

示例（`/compact` 命令）：

```typescript
// src/commands/compact/index.ts
// local-jsx 类型
// call(onDone, context, args) 中手动解析 args
// 支持 --full, --auto 等标志
```

### 4.2 自动补全

命令注册系统提供：
- `name` — 命令名（如 `compact`）
- `aliases` — 别名
- `description` — 描述文本
- `argumentHint` — 参数提示（灰色显示）
- `whenToUse` — 使用场景描述

这些信息用于 REPL 的自动补全和建议系统。

---

## 5. 命令的权限控制

### 5.1 可见性控制层级

```
1. availability — 认证/提供商要求（静态）
   ↓ meetsAvailabilityRequirement()
2. isEnabled() — 运行时开关（GrowthBook feature flag 等）
   ↓
3. isHidden — 是否在帮助中隐藏
   ↓
4. 内部/外部构建 — INTERNAL_ONLY_COMMANDS 只在 ant 构建中包含
```

### 5.2 命令执行权限

- **`supportsNonInteractive`**（LocalCommand）：是否在非交互模式下可用
- **`disableModelInvocation`**：是否禁止 LLM 调用此命令
- **`userInvocable`**：用户是否可以通过输入命令名调用
- **`isSensitive`**：敏感命令，参数在对话历史中脱敏
- **`immediate`**：是否立即执行（跳过队列）

### 5.3 权限命令

`/permissions` 命令（`src/commands/permissions/`）直接管理工具的权限规则：

- 查看 `alwaysAllow` / `alwaysDeny` / `alwaysAsk` 规则
- 添加/删除权限规则
- 这些规则影响 `Tool.checkPermissions()` 和 `canUseTool()` 的行为

---

## 6. 模块架构图

```
                    ┌──────────────────────┐
                    │     用户输入 /xxx     │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │   命令匹配与分发      │
                    │  findCommand(name)    │
                    └──────────┬───────────┘
                               │
              ┌────────────────┼─────────────────┐
              │                │                  │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌───────▼──────────┐
    │  PromptCommand  │ │ LocalCommand │ │ LocalJSXCommand  │
    │  type: 'prompt' │ │ type:'local' │ │ type: 'local-jsx'│
    └─────────┬──────┘ └──────┬───────┘ └───────┬──────────┘
              │               │                  │
    ┌─────────▼──────┐ ┌──────▼───────┐ ┌───────▼──────────┐
    │getPromptForCmd()│ │ load().call()│ │load().call(       │
    │ → 注入对话      │ │ → 文本结果   │ │  onDone, ctx, args│
    │ → LLM 调用工具  │ │              │ │ ) → React UI      │
    └────────────────┘ └──────────────┘ └───────────────────┘

    ┌─────────────────────────────────────────────────────┐
    │                 命令来源                              │
    │  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐ │
    │  │内置命令   │ │技能目录   │ │插件技能 │ │MCP prompts│ │
    │  │commands/ │ │.claude/  │ │plugins │ │MCP server│ │
    │  │          │ │skills/   │ │        │ │          │ │
    │  └──────────┘ └──────────┘ └────────┘ └──────────┘ │
    └─────────────────────────────────────────────────────┘
```

---

## 7. 关键代码示例

### 7.1 `/init` 命令 — PromptCommand

```typescript
// src/commands/init.ts
// 这是一个 PromptCommand，生成 CLAUDE.md 的 prompt
const initCommand: Command = {
  type: 'prompt',
  name: 'init',
  description: 'Initialize CLAUDE.md for your project',
  progressMessage: 'Analyzing codebase',
  contentLength: 0,
  source: 'builtin',
  async getPromptForCommand(args, context) {
    // 返回包含详细指令的 prompt 文本
    // LLM 收到后会使用 Read、Write 等工具完成 CLAUDE.md 生成
    return [{ type: 'text', text: NEW_INIT_PROMPT }]
  },
}
```

### 7.2 `/help` 命令 — LocalJSXCommand

```typescript
// src/commands/help/index.ts
const help = {
  type: 'local-jsx',
  name: 'help',
  description: 'Show help and available commands',
  load: () => import('./help.js'),  // 懒加载
} satisfies Command
```

### 7.3 技能目录加载

```typescript
// src/skills/loadSkillsDir.ts
// 扫描 .claude/skills/ 目录，为每个技能创建 PromptCommand
export async function getSkillDirCommands(cwd: string): Promise<Command[]> {
  // 扫描目录 → 读取 skill.md → 创建 PromptCommand
  // name = 目录名, source = 'skills'
  // getPromptForCommand() 返回 skill.md 内容
}
```

---

## 8. 设计优缺点分析

### 优点

1. **统一命令接口**：三种命令类型（prompt/local/local-jsx）覆盖了所有使用场景
2. **懒加载设计**：所有命令通过 `load()` 延迟导入，减少启动时间
3. **多源命令**：支持内置命令、技能目录、插件、MCP 等多种命令来源
4. **Prompt 命令模式**：将斜杠命令转化为 LLM prompt，巧妙复用了 LLM 的能力
5. **Feature Flag DCE**：通过 `feature()` 实现构建时消除不需要的命令
6. **可用性分层**：`availability` + `isEnabled()` + `isHidden` 三层控制命令可见性

### 缺点

1. **`commands.ts` 注册文件过长**：需要手动维护 100+ 命令的导入和注册
2. **无自动发现机制**：新增命令需要手动修改 `commands.ts`，容易遗漏
3. **参数解析不统一**：每个命令自行解析参数字符串，缺乏一致的参数解析框架
4. **类型系统复杂**：`Command` 是联合类型，需要频繁的类型收窄
5. **Feature Flag 耦合**：命令的可用性高度依赖 `feature()` 和 `process.env`，测试困难
6. **技能命令与内置命令边界模糊**：技能目录的命令和内置命令混在一起管理
7. **错误处理不一致**：local 和 local-jsx 命令的错误处理方式不同
