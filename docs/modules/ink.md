# Ink — 终端渲染框架适配层

## 模块概览

| 指标 | 值 |
|------|-----|
| 文件数 | 96 |
| 总代码行数 | ~13,306 |
| 核心职责 | 终端 UI 渲染引擎、布局系统、事件处理、自定义 Hooks |
| 关键文件 | `ink.tsx`, `reconciler.ts`, `renderer.ts`, `hooks/use-input.ts`, `layout/yoga.ts` |

### 关键文件清单

| 文件 | 行数 | 职责 |
|------|------|------|
| `ink.tsx` | 1,722 | 核心渲染类（Ink 主类） |
| `reconciler.ts` | — | React Reconciler 适配 |
| `renderer.ts` | — | 渲染器抽象 |
| `Ansi.tsx` | 291 | ANSI 转义序列处理 |
| `hooks/use-input.ts` | — | 键盘输入 Hook |
| `hooks/use-terminal-viewport.ts` | — | 视口适配 Hook |
| `layout/yoga.ts` | — | Yoga 布局引擎集成 |
| `events/` | 10 文件 | 事件系统 |

### 目录结构

```
ink/
├── ink.tsx              — 核心渲染主类
├── reconciler.ts        — React Reconciler 适配
├── renderer.ts          — 渲染器抽象
├── render-node-to-output.ts — 节点到输出渲染
├── render-to-screen.ts  — 屏幕渲染 + 搜索高亮
├── render-border.ts     — 边框渲染
├── dom.ts               — 虚拟 DOM 定义
├── output.ts            — 输出缓冲
├── colorize.ts          — 颜色处理
├── optimizer.ts         — 渲染优化
├── screen.ts            — 屏幕缓冲区
├── selection.ts         — 文本选择
├── searchHighlight.ts   — 搜索高亮
├── log-update.ts        — 日志更新（diff 渲染）
├── frame.ts             — 帧管理
├── focus.ts             — 焦点管理
├── hit-test.ts          — 点击/悬停检测
├── instances.ts         — 实例管理
├── node-cache.ts        — 节点缓存
├── measure-text.ts      — 文本测量
├── measure-element.ts   — 元素测量
├── line-width-cache.ts  — 行宽缓存
├── get-max-width.ts     — 最大宽度计算
├── parse-keypress.ts    — 按键解析
├── wrap-text.ts         — 文本换行
├── wrapAnsi.ts          — ANSI 换行
├── clearTerminal.ts     — 终端清除
├── constants.ts         — 常量
├── Ansi.tsx             — ANSI 组件
│
├── hooks/               — 自定义 Hooks
│   ├── use-animation-frame.ts
│   ├── use-app.ts
│   ├── use-declared-cursor.ts
│   ├── use-input.ts
│   ├── use-interval.ts
│   ├── use-search-highlight.ts
│   ├── use-selection.ts
│   ├── use-stdin.ts
│   ├── use-tab-status.ts
│   ├── use-terminal-focus.ts
│   ├── use-terminal-title.ts
│   └── use-terminal-viewport.ts
│
├── layout/              — 布局引擎
│   ├── engine.ts        — 布局引擎抽象
│   ├── geometry.ts      — 几何计算
│   ├── node.ts          — 布局节点
│   └── yoga.ts          — Yoga 集成
│
├── events/              — 事件系统
│   ├── event.ts         — 基础事件
│   ├── dispatcher.ts    — 事件分发器
│   ├── emitter.ts       — 事件发射器
│   ├── event-handlers.ts — 事件处理器
│   ├── click-event.ts
│   ├── focus-event.ts
│   ├── input-event.ts
│   ├── keyboard-event.ts
│   ├── terminal-event.ts
│   └── terminal-focus-event.ts
│
├── components/          — 内置组件
│   └── App.js           — 根组件
│
└── termio/              — 终端 I/O 控制
    ├── csi.ts           — CSI 序列
    ├── dec.ts           — DEC 私有序列
    └── osc.ts           — OSC 序列
```

---

## 1. Ink 核心类

**文件路径**: `src/ink/ink.tsx` (1,722 行)

### 架构设计

```
Ink (主类)
├── 构造时:
│   ├── 创建 Terminal 适配器
│   ├── 创建 React Reconciler
│   ├── 创建 FocusManager
│   ├── 创建 LogUpdate (diff 输出)
│   ├── 创建双缓冲 (frontFrame / backFrame)
│   └── 注册 TTY 事件处理
│
├── 渲染循环:
│   ├── scheduleRender() — 节流调度
│   ├── rebuild() — Yoga 布局计算
│   ├── render() — 渲染到输出缓冲
│   ├── diff + writeDiffToTerminal() — 增量写入终端
│   └── 帧交换 (frontFrame ↔ backFrame)
│
├── 事件处理:
│   ├── stdin → parseKeypress → KeyboardEvent
│   ├── mouse events → click/hover dispatch
│   ├── resize → 重新布局 + 重渲染
│   └── terminal focus/unfocus
│
└── 生命周期:
    ├── render(ReactNode) — 挂载/更新组件树
    ├── unmount() — 卸载
    ├── waitUntilExit() — 等待退出
    └── cleanup() — 恢复终端状态
```

### 核心代码

```typescript
export default class Ink {
  private readonly terminal: Terminal
  private scheduleRender: (() => void) & { cancel?: () => void }
  private readonly container: FiberRoot
  private readonly focusManager: FocusManager
  private renderer: Renderer
  private frontFrame: Frame
  private backFrame: Frame

  // 终端尺寸
  private terminalColumns: number
  private terminalRows: number

  constructor({ stdout, stdin, stderr, exitOnCtrlC, patchConsole, waitUntilExit, onFrame }: Options) {
    this.terminal = { stdout, stdin, stderr, columns, rows }
    this.container = reconciler.createContainer(rootNode, ConcurrentRoot, null, false, null, '', onError, null)
    this.focusManager = new FocusManager()
    this.frontFrame = emptyFrame()
    this.backFrame = emptyFrame()
    // ...
  }
}
```

**关键设计**:
- **双缓冲帧**: `frontFrame` (当前屏幕) 和 `backFrame` (下一帧) 避免闪烁
- **React Concurrent Mode**: 使用 `ConcurrentRoot` 支持并发特性
- **增量渲染**: `writeDiffToTerminal()` 只写入变化部分，减少终端 I/O

---

## 2. 布局系统（Yoga）

**文件路径**: `src/ink/layout/`

### 架构

```
Yoga 布局引擎 (Facebook 的 Flexbox 实现)
├── engine.ts — 布局引擎抽象（原生/JS 回退）
├── node.ts   — 布局节点包装
├── geometry.ts — 几何计算
└── yoga.ts   — Yoga Wasm/Native 绑定
```

```
布局流程:
React 组件树 → reconciler → DOM 节点
    ↓
Yoga 节点树 (Flexbox 属性)
    ↓
Yoga 计算布局 (width, height, x, y)
    ↓
render-node-to-output() — 按布局位置渲染到输出缓冲
```

**性能优化**:
- 原生 Yoga 绑定（通过 `getYogaCounters()` 追踪性能）
- 布局结果缓存，仅在节点变化时重新计算
- `optimizer.ts` 跳过未变化的子树

---

## 3. 自定义 Hooks

### 3.1 use-input — 键盘输入

```typescript
// use-input.ts — 注册键盘事件处理器
// 支持:
// - 普通按键 (a-z, 0-9, etc.)
// - 功能键 (Enter, Escape, Tab, etc.)
// - 修饰键组合 (Ctrl+C, Alt+Backspace, etc.)
// - 扩展键协议 (Kitty keyboard protocol)
```

### 3.2 use-terminal-viewport — 视口适配

响应终端尺寸变化，提供视口信息：

```typescript
// hooks/use-terminal-viewport.ts
// 监听终端 resize 事件
// 提供当前 columns/rows
// 通知组件重新布局
```

### 3.3 其他 Hooks

| Hook | 职责 |
|------|------|
| `use-animation-frame` | 动画帧回调 |
| `use-app` | 应用生命周期（exit, stdin） |
| `use-declared-cursor` | 光标声明管理 |
| `use-interval` | 定时器 |
| `use-search-highlight` | 搜索高亮状态 |
| `use-selection` | 文本选择（鼠标/键盘） |
| `use-stdin` | stdin 读取 |
| `use-tab-status` | Tab 状态栏 |
| `use-terminal-focus` | 终端焦点检测 |
| `use-terminal-title` | 终端标题设置 |

---

## 4. 事件系统

**文件路径**: `src/ink/events/`

```
事件分发架构:
stdin raw data
    ↓
parse-keypress.ts → ParsedKey
    ↓
KeyboardEvent (事件对象)
    ↓
dispatcher.ts → 分发到注册的处理器
    ↓
use-input / use-app 等 Hooks 接收事件

终端事件:
├── keyboard-event.ts  — 键盘事件
├── click-event.ts     — 鼠标点击事件
├── focus-event.ts     — 焦点事件
├── input-event.ts     — 通用输入事件
├── terminal-event.ts  — 终端事件（resize等）
└── terminal-focus-event.ts — 终端焦点事件
```

### 事件处理器链

```typescript
// events/event-handlers.ts
// 管理事件处理器的注册和分发
// 支持优先级和事件拦截
```

---

## 5. 渲染管线

```
React 组件树
    ↓ (React Reconciler)
Ink DOM 节点树
    ↓ (Yoga Layout)
布局坐标计算
    ↓ (render-node-to-output)
Output 缓冲区 (行×列的 Cell 矩阵)
    ↓ (optimizer)
渲染优化 (跳过未变化区域)
    ↓ (render-to-screen)
Screen Diff → 增量更新
    ↓ (writeDiffToTerminal)
终端 ANSI 输出
```

### Screen 缓冲区

```typescript
// screen.ts — 高效的屏幕缓冲区
// Cell 结构: { char, style, hyperlink }
// StylePool / CharPool / HyperlinkPool — 对象池复用
// writeDiffToTerminal() — 增量 diff 写入
```

---

## 6. 终端 I/O 控制

### CSI 序列 (csi.ts)
- 光标移动、定位
- 行清除/插入/删除
- 滚动区域设置

### DEC 私有序列 (dec.ts)
```typescript
export const ENTER_ALT_SCREEN = '\x1b[?1049h'
export const EXIT_ALT_SCREEN = '\x1b[?1049l'
export const SHOW_CURSOR = '\x1b[?25h'
export const ENABLE_MOUSE_TRACKING = '\x1b[?1000h'
export const DISABLE_MOUSE_TRACKING = '\x1b[?1000l'
```

### OSC 序列 (osc.ts)
- 剪贴板操作
- Tab 状态栏
- ITerm2 进度条
- 终端提示集成

---

## 7. 文本选择与搜索

### 选择系统 (selection.ts)
- 鼠标选择（单击、双击选词、三击选行）
- 键盘选择（Shift+方向键扩展）
- 纯文本 URL 检测
- 选择文本复制到剪贴板

### 搜索高亮 (searchHighlight.ts)
- 正则匹配位置扫描
- 高亮叠加渲染
- 当前匹配高亮（vs 其他匹配）

---

## 8. 焦点管理 (focus.ts)

```
FocusManager
├── 焦点环 (focus ring) — Tab 循环
├── 焦点进入/离开事件
├── 方向键焦点导航
└── 嵌套焦点域
```

---

## 9. 设计优缺点分析

### 优点

1. **高性能渲染**: 双缓冲 + 增量 diff + 对象池，最小化终端 I/O
2. **React 生态集成**: 使用 React Reconciler，组件开发体验与 React 一致
3. **Yoga 布局**: Flexbox 布局算法，支持复杂 UI 排列
4. **丰富的事件系统**: 支持键盘、鼠标、焦点、终端等多种事件
5. **终端兼容性**: Alt Screen、Kitty 键盘协议、扩展键等现代终端特性
6. **文本选择**: 完整的鼠标/键盘选择支持，超越一般 TUI 框架

### 缺点

1. **实现复杂度高**: 96 个文件，核心类 Ink 达 1,722 行
2. **Yoga 原生依赖**: 需要平台特定的 Yoga 绑定，增加构建复杂度
3. **Screen 缓冲区管理**: 手动管理 Cell 矩阵，内存使用需要仔细优化
4. **并发渲染**: React Concurrent Mode 在终端环境中的边界情况需要特殊处理
5. **调试困难**: 双缓冲 + 增量 diff 的渲染管线不易调试

---

## 10. 与标准 Ink 的差异

这是 Claude Code 对 [Ink](https://github.com/vadimdemedes/ink) 框架的深度定制版本：

| 特性 | 标准 Ink | Claude Code Ink |
|------|---------|-----------------|
| 布局引擎 | Yoga (Wasm) | Yoga (Native + Wasm 回退) |
| 渲染模式 | 全帧重绘 | 双缓冲增量 diff |
| 鼠标支持 | 基础 | 完整（选择、悬停、点击） |
| 搜索高亮 | 无 | 内置正则搜索 + 高亮 |
| 文本选择 | 无 | 鼠标 + 键盘选择 |
| 事件系统 | 简单 | 完整分发器 + 处理器链 |
| 终端特性 | 基础 | Alt Screen、Kitty 协议、Tab Status |
| 性能追踪 | 无 | Yoga 计数器、帧时间 |
| React 版本 | Legacy | Concurrent Mode |
