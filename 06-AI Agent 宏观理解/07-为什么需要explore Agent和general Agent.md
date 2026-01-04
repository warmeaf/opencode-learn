# 为什么需要 explore 和 general Agent

## 核心原因

OpenCode 需要两个子 Agent 是基于**职责分离**和**优化性能**的设计原则。它们各自专注不同类型的工作场景。

---

## Agent 对比

| 特性           | explore Agent                          | general Agent                        |
| -------------- | -------------------------------------- | ------------------------------------ |
| **核心定位**   | 代码搜索专家                           | 通用任务执行者                       |
| **主要用途**   | 快速查找文件、搜索代码、理解代码库结构 | 执行复杂多步骤任务、并行处理工作单元 |
| **权限**       | 搜索+只读（bash 限文件操作）           | 可执行操作（除 task 和 todo 工具）   |
| **专用提示词** | ✅ 有搜索优化提示                      | ❌ 使用默认提示                      |
| **可见性**     | 用户可调用                             | hidden（内部使用）                   |

---

## explore Agent 的设计意图

### 1. 专业化搜索能力

explore Agent 配备了专门的系统提示词：

```
You are a file search specialist. You excel at thoroughly navigating and exploring codebases.
```

这个提示词引导 Agent：

- 使用 Glob 进行广泛文件匹配
- 使用 Grep 进行正则表达式搜索
- 使用 Read 读取特定文件内容
- 使用 Bash 进行文件列表、复制等操作
- 使用 CodeSearch 进行代码搜索
- 根据调用者指定的深度（quick/medium/very thorough）调整搜索策略

### 2. 严格只读保护

权限配置明确拒绝所有编辑操作：

```typescript
permission: {
  "*": "deny",        // 默认拒绝所有
  grep: "allow",      // 只允许搜索相关
  glob: "allow",
  list: "allow",
  bash: "allow",      // 仅用于文件操作
  webfetch: "allow",
  websearch: "allow",
  codesearch: "allow",
  read: "allow",
}
```

系统提示词进一步限制 bash 使用：

```
Do not run bash commands that modify the user's system state in any way
```

这确保搜索过程不会意外修改代码库。

### 3. 性能优化

专注于快速搜索，避免了：

- 不需要的工具调用
- 编辑文件的开销
- 复杂的决策过程

---

## general Agent 的设计意图

### 1. 并行多任务处理

描述明确指出其能力：

```
General-purpose agent for researching complex questions and executing multi-step tasks.
Use this agent to execute multiple units of work in parallel.
```

当主 Agent 需要同时处理多个独立任务时，会调用 general Agent 来并行执行。

### 2. 防止无限嵌套

权限配置禁用了 todo 工具：

```typescript
permission: {
  todoread: "deny",
  todowrite: "deny",
}
```

**真正的 task 禁用在 session 层面实现**：

当调用 Task 工具创建子 session 时，会显式添加禁用规则：

```typescript
permission: [
  {
    permission: "task",
    pattern: "*",
    action: "deny",
  },
  // ...
]
```

这防止子 Agent 再调用子 Agent，避免无限嵌套和资源浪费。

### 3. 通用任务执行

具备完整的操作能力（除 task 和 todo 外）：

- 读写文件（Read、Write、Edit）
- 执行 bash 命令（Bash）
- 网络请求（Webfetch）
- 代码搜索（Grep、Glob、CodeSearch）
- 其他所有工具操作

---

## 实际使用场景

### 场景 1：代码库探索

用户问："找出所有 API 端点定义"

**主 Agent 的决策**：

```
这是一个搜索任务 → 调用 explore Agent，thoroughness="very thorough"
```

**explore Agent 执行**：

1. 使用 Glob 查找 `**/*route*.{ts,tsx,js,jsx}`
2. 使用 Grep 搜索 `/api/` 或 `.route(` 模式
3. 读取关键文件验证
4. 返回完整路径列表

---

### 场景 2：并行文件处理

用户问："重构三个不同的模块，然后运行测试"

**主 Agent 的决策**：

```
这是多个独立任务 → 调用 general Agent 3 次，并行执行
```

**general Agent 执行**：

- 每个实例在独立会话中运行
- 使用 Edit、Bash 等工具完成任务
- 返回结果给主 Agent

---

## 为什么不合并成一个？

### 如果只用 explore Agent

❌ 无法执行编辑操作
❌ 无法并行处理复杂任务
❌ 无法运行测试或构建

### 如果只用 general Agent

❌ 搜索时可能意外修改文件
❌ 搜索性能不如专用 Agent
❌ 搜索提示词不够优化

### 两者的协同

主 Agent 负责整体规划：

- 需要"找文件" → 调用 explore
- 需要"改代码" → 调用 general
- 需要"同时做 3 件事" → 调用 3 个 general

---

## 代码位置

- **Agent 定义**：`packages/opencode/src/agent/agent.ts:77-115`
- **explore 提示词**：`packages/opencode/src/agent/prompt/explore.txt`
- **Task 工具实现**：`packages/opencode/src/tool/task.ts`

---

## 总结

explore 和 general Agent 的存在体现了 Agent 系统的**专业化分工**：

- **explore**：专注、快速、安全地搜索代码库
- **general**：灵活、并行、可靠地执行复杂任务

这种设计让主 Agent 可以像项目经理一样，根据任务特性选择最合适的工具，提高了整个系统的效率、安全性和可维护性。
