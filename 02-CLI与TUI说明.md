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
| 特性 | CLI | TUI |
|------|-----|-----|
| 交互方式 | 纯命令输入 | 菜单、按钮、快捷键 |
| 视觉效果 | 纯文本流 | 布局化的界面 |
| 学习曲线 | 需要记忆命令 | 更直观易用 |
| 复杂度 | 简单 | 较复杂 |

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

## 对比总结

### 三种界面类型对比

| 界面类型 | 英文 | 交互方式 | 例子 |
|---------|------|---------|------|
| CLI | Command Line Interface | 命令行文本输入 | bash, cmd, git |
| TUI | Text-based User Interface | 终端内的类图形界面 | htop, vim, tmux |
| GUI | Graphical User Interface | 图形窗口鼠标操作 | VS Code, Chrome |

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
