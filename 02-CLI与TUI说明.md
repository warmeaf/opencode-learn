# CLI 和 TUI 说明

## 什么是 CLI (Command Line Interface)？

**CLI** 是**命令行界面**（Command Line Interface）的缩写。

### 定义

CLI 是一种基于文本的用户界面，用户通过输入命令与计算机程序进行交互。

### 特点

- **纯文本交互**：用户通过键盘输入文本命令，程序以文本形式返回结果
- **无图形界面**：没有窗口、按钮、图标等图形元素
- **高效快捷**：对于熟练用户来说，执行速度非常快
- **资源占用少**：不需要渲染图形界面，系统资源占用较低
- **适合脚本化**：可以轻松编写脚本自动化执行任务

### 常见例子

- Windows 的 `cmd.exe` 或 PowerShell
- Linux/macOS 的 Terminal（Bash、Zsh 等）
- Git 命令：`git status`、`git commit`
- Node.js 的 `npm` 命令
- 本项目使用的 Claude Code 也是 CLI 工具

### 使用场景

```bash
# 文件操作
ls -la
mkdir myfolder
cp file1.txt file2.txt

# Git 操作
git add .
git commit -m "update"
git push

# 包管理
npm install
pip install requests
```

---

## 什么是 TUI (Text-based User Interface)？

**TUI** 是**文本用户界面**（Text-based User Interface）的缩写，也被称为 Terminal User Interface。

### 定义

TUI 是一种运行在终端中的用户界面，虽然使用文本字符，但通过布局和颜色模拟图形界面的效果。

### 特点

- **终端内运行**：在命令行终端中运行，但比普通 CLI 更美观
- **类似图形界面**：有菜单、按钮、窗口等元素（用字符绘制）
- **鼠标支持**：通常支持鼠标点击操作
- **颜色丰富**：使用颜色区分不同元素，提高可读性
- **交互性强**：可以使用键盘快捷键、方向键导航

### 与 CLI 的区别

| 特性     | CLI          | TUI                |
| -------- | ------------ | ------------------ |
| 交互方式 | 纯命令输入   | 菜单、按钮、快捷键 |
| 视觉效果 | 纯文本流     | 布局化的界面       |
| 学习曲线 | 需要记忆命令 | 更直观易用         |
| 复杂度   | 简单         | 较复杂             |

### 常见例子

- **htop**：系统进程监控工具（带有图形化的进度条）
- **vim/nano**：文本编辑器（有菜单栏和状态栏）
- **tmux**：终端复用器（分屏、窗口管理）
- **npm init**：交互式创建 package.json
- **git rebase -i**：交互式变基

### TUI 示意图

```
┌─────────────────────────────────────┐
│  File  Edit  View  Help              │  ← 菜单栏
├─────────────────────────────────────┤
│                                     │
│  ┌─ Process List ──────────────┐   │
│  │ [+] PID   CPU%   MEM%       │   │  ← 内容区
│  │     123   5.2    2.1        │   │
│  │     456   12.8   8.4        │   │
│  │                              │   │
│  └──────────────────────────────┘   │
│                                     │
│  [Save]  [Cancel]  [Help]           │  ← 按钮
│                                     │
└─────────────────────────────────────┘
│ Press 'q' to quit                    │  ← 状态栏
└─────────────────────────────────────┘
```

---

## TUI 渲染原理详解

OpenCode 的 TUI 使用了 **@opentui/core** 和 **@opentui/solid** 框架，采用类似于 React/Vue 的组件化思想，但针对终端环境进行了专门优化。

### 技术栈

| 技术               | 版本   | 用途         |
| ------------------ | ------ | ------------ |
| **@opentui/core**  | 0.1.63 | TUI 核心框架 |
| **@opentui/solid** | 0.1.63 | SolidJS 集成 |
| **solid-js**       | 1.9.9  | 响应式框架   |

### 核心原理

#### 1. 基于 Solid.js 的响应式系统

TUI 使用 Solid.js 的响应式系统来管理状态和 UI 更新：

```typescript
// 主应用入口 (packages/opencode/src/cli/cmd/tui/app.tsx)
render(() => {
  return (
    <ErrorBoundary>
      <ArgsProvider {...input.args}>
        <ExitProvider onExit={onExit}>
          <ThemeProvider mode={mode}>
            <RouteProvider>
              <App />
            </RouteProvider>
          </ThemeProvider>
        </ExitProvider>
      </ArgsProvider>
    </ErrorBoundary>
  )
}, {
  targetFps: 60,  // 60 FPS 渲染目标
  gatherStats: false,
  exitOnCtrlC: false,
  useKittyKeyboard: {},
})
```

**特点**：

- **细粒度响应式**：只有依赖的数据变化时才更新
- **60 FPS 渲染**：确保流畅的交互体验
- **组件化架构**：可复用的 UI 组件

#### 2. Renderable 组件系统

OpenCode TUI 定义了一套专门针对终端的组件：

| 组件                    | 用途         |
| ----------------------- | ------------ |
| **BoxRenderable**       | 基础布局容器 |
| **ScrollBoxRenderable** | 可滚动区域   |
| **TextareaRenderable**  | 文本输入框   |
| **InputRenderable**     | 单行输入框   |
| **TextRenderable**      | 文本显示     |

**使用示例**：

```typescript
// 获取组件实例引用
let input: TextareaRenderable
let scroll: ScrollBoxRenderable

// 使用 ref 绑定
<TextareaRenderable ref={(val: TextareaRenderable) => (input = val)} />
<ScrollBoxRenderable ref={(r: ScrollBoxRenderable) => (scroll = r)} />

// 通过实例控制组件
input.focus()
scroll.scrollToBottom()
```

#### 3. ANSI Escape Codes 终端控制

TUI 通过特殊的转义序列来控制终端的显示：

| 功能       | ANSI 转义码      | 说明               |
| ---------- | ---------------- | ------------------ |
| 光标定位   | `\x1b[row;colH`  | 移动光标到指定位置 |
| 文本颜色   | `\x1b[38;5;255m` | 设置前景色         |
| 背景颜色   | `\x1b[48;5;255m` | 设置背景色         |
| 重置属性   | `\x1b[0m`        | 重置所有属性       |
| 加粗       | `\x1b[1m`        | 粗体文本           |
| 下划线     | `\x1b[4m`        | 下划线文本         |
| 查询背景色 | `\x1b]11;?\x07`  | 获取终端背景色     |

**示例**：

```typescript
// 设置终端标题
renderer.setTerminalTitle("OpenCode")

// 查询终端背景色（检测亮/暗模式）
process.stdout.write("\x1b]11;?\x07")

// 写入 ANSI 转义码
renderer.writeOut("\x1b[38;5;255mHello\x1b[0m")
```

#### 4. 渲染器管理

通过 `useRenderer()` hook 获取渲染器实例：

```typescript
const renderer = useRenderer()

// 常用渲染器操作
renderer.requestRender() // 请求重新渲染
renderer.setTerminalTitle() // 设置终端标题
renderer.suspend() // 暂停渲染（调用外部编辑器时）
renderer.resume() // 恢复渲染
renderer.getSelection() // 获取选中文本
renderer.clearSelection() // 清除选择
renderer.writeOut(escapeCode) // 直接写入 ANSI 代码
renderer.toggleDebugOverlay() // 切换调试覆盖层
```

**实际使用场景**：

```typescript
// 调用外部编辑器前暂停渲染
renderer.suspend()
renderer.currentRenderBuffer.clear()
// ... 打开编辑器
renderer.currentRenderBuffer.clear()
renderer.resume()
renderer.requestRender()
```

#### 5. 事件驱动架构

TUI 支持键盘和鼠标事件：

```typescript
// 键盘事件处理
useKeyboard((evt) => {
  if (evt.ctrl && evt.name === "c") {
    // 处理 Ctrl+C
  }
  if (evt.name === "enter") {
    // 处理 Enter 键
  }
})

// 鼠标事件处理
<BoxRenderable
  onMouseUp={() => {
    // 鼠标点击事件
    const text = renderer.getSelection()?.getSelectedText()
    if (text) {
      // 复制到剪贴板
      Clipboard.copy(text)
    }
  }}
/>
```

#### 6. 流式更新机制

通过 SSE (Server-Sent Events) 实时接收 AI 响应并更新 UI：

```typescript
// 监听消息更新事件
sdk.event.on(MessageV2.Event.PartUpdated.type, (evt) => {
  // 实时更新 part 内容
  switch (evt.part.type) {
    case "text":
      // 更新文本内容
      break
    case "tool":
      // 更新工具调用状态
      break
    case "reasoning":
      // 更新推理过程
      break
  }
})
```

### 技术栈层次

```
┌────────────────────────────────────┐
│  Solid.js 组件 (tsx)               │
│  - 响应式数据流                      │
│  - 组件生命周期                      │
│  - JSX 语法                         │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│  @opentui/solid 集成层              │
│  - render() 函数                    │
│  - useRenderer() hook               │
│  - useKeyboard() hook               │
│  - useTerminalDimensions() hook     │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│  @opentui/core 核心层               │
│  - Renderable 组件系统               │
│  - 事件处理 (键盘/鼠标)               │
│  - ANSI 转义码生成                   │
│  - 布局引擎 (Yoga Layout)           │
└────────────────────────────────────┘
           ↓
┌────────────────────────────────────┐
│  终端 (Terminal)                   │
│  - 解析 ANSI 转义码                 │
│  - 渲染文本和布局                    │
│  - 处理键盘/鼠标输入                 │
└────────────────────────────────────┘
```

### 关键特性

#### 1. 60 FPS 渲染目标

- 确保流畅的交互体验
- 使用 `requestRender()` 进行增量更新

#### 2. 细粒度更新

- 只更新变化的部分
- 减少不必要的重绘

#### 3. 键盘增强协议

- 支持 Kitty 键盘协议
- 处理复杂的键盘组合

#### 4. 鼠标支持

- 点击、选择、滚动等操作
- 复制到剪贴板（OSC52 协议）

#### 5. 颜色主题

- 自动检测终端亮/暗模式
- 通过 ANSI 转义码查询背景色

#### 6. 文本选择

- 支持文本选择和复制
- 使用 OSC52 协议跨终端复制

### 实际应用示例

#### 检测终端亮/暗模式

```typescript
async function getTerminalBackgroundColor(): Promise<"dark" | "light"> {
  return new Promise((resolve) => {
    const handler = (data: Buffer) => {
      const str = data.toString()
      const match = str.match(/\x1b]11;([^\x07\x1b]+)/)
      if (match) {
        const color = match[1]
        // 解析 RGB 值并计算亮度
        const r = parseInt(color.substring(1, 3), 16)
        const g = parseInt(color.substring(3, 5), 16)
        const b = parseInt(color.substring(5, 7), 16)
        const luminance = (0.299 * r + 0.587 * g + 0.114 * b) / 255
        resolve(luminance > 0.5 ? "light" : "dark")
      }
    }
    process.stdin.setRawMode(true)
    process.stdin.on("data", handler)
    process.stdout.write("\x1b]11;?\x07") // 查询背景色
  })
}
```

#### 响应式布局

```typescript
const dimensions = useTerminalDimensions()

<BoxRenderable
  width={dimensions().width}
  height={dimensions().height}
  backgroundColor={theme.background}
>
  <Switch>
    <Match when={route.data.type === "home"}>
      <Home />
    </Match>
    <Match when={route.data.type === "session"}>
      <Session />
    </Match>
  </Switch>
</BoxRenderable>
```

### 相关文件位置

- **主应用**: `packages/opencode/src/cli/cmd/tui/app.tsx`
- **会话页面**: `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`
- **输入组件**: `packages/opencode/src/cli/cmd/tui/component/prompt/`
- **对话框**: `packages/opencode/src/cli/cmd/tui/ui/`
- **上下文**: `packages/opencode/src/cli/cmd/tui/context/`

### 参考资源

- [OpenTUI GitHub](https://github.com/sst/opentui)
- [ANSI Escape Codes](https://en.wikipedia.org/wiki/ANSI_escape_code)
- [Solid.js 官方文档](https://www.solidjs.com/docs)

---

## 对比总结

### 三种界面类型对比

| 界面类型 | 英文                      | 交互方式           | 例子            |
| -------- | ------------------------- | ------------------ | --------------- |
| CLI      | Command Line Interface    | 命令行文本输入     | bash, cmd, git  |
| TUI      | Text-based User Interface | 终端内的类图形界面 | htop, vim, tmux |
| GUI      | Graphical User Interface  | 图形窗口鼠标操作   | VS Code, Chrome |

### 选择建议

- **使用 CLI 当**：
  - 需要快速执行简单命令
  - 编写自动化脚本
  - 服务器远程操作
  - 熟悉命令，追求效率

- **使用 TUI 当**：
  - 需要查看复杂信息（如系统监控）
  - 需要交互式操作（如配置向导）
  - 在终端中需要更好的视觉体验
  - 需要同时使用多个功能（如分屏编辑）

## 参考资源

- [Command-line interface - Wikipedia](https://en.wikipedia.org/wiki/Command-line_interface)
- [Text-based user interface - Wikipedia](https://en.wikipedia.org/wiki/Text-based_user_interface)
- Awesome CLI 列表: https://github.com/agarrharr/awesome-cli-apps
- Awesome TUI 列表: https://github.com/rothgar/awesome-tuis

---

**文档创建时间**: 2025-12-29
**作者**: Claude Code
**用途**: 学习笔记 - CLI 和 TUI 概念说明
