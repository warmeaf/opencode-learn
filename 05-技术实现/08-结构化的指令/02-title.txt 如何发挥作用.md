# title.txt 提示词如何发挥作用

## 概述

`title.txt` 是一个专门用于生成对话标题的系统提示词，它指导 AI 模型为用户会话创建简洁、有意义的标题，帮助用户后续查找和管理对话历史。

---

## 提示词内容分析

位置：`packages/opencode/src/agent/prompt/title.txt`

该提示词定义了一个标题生成器的行为规范，包含以下核心部分：

### 1. 身份定义

```
You are a title generator. You output ONLY a thread title. Nothing else.
```

### 2. 任务要求

- 生成能帮助用户后续检索对话的标题
- 遵循所有规则
- 使用示例作为参考
- 输出格式：单行、≤50 字符、无解释

### 3. 生成规则

- **聚焦主题**：关注用户需要检索的主要话题或问题
- **使用动名词**：使用 -ing 形式的动词（Debugging、Implementing、Analyzing）
- **保持精确**：保留技术术语、数字、文件名、HTTP 状态码等
- **移除冗余**：删除 the、this、my、a、an 等词
- **不假设技术栈**：不在标题中推断使用的技术
- **不使用工具**：标题中不提及任何工具
- **仅生成标题**：不回答问题，只生成标题
- **避免元语言**：标题中不包含"summarizing"或"generating"等元词汇
- **始终输出**：即使输入极简也要输出有意义的标题
- **处理简短消息**：对于问候语等，生成反映用户意图的标题（如 Greeting、Quick check-in 等）

### 4. 示例参考

```
"debug 500 errors in production" → Debugging production 500 errors
"refactor user service" → Refactoring user service
"why is app.js failing" → Analyzing app.js failure
```

---

## 在代码库中的应用

### 1. Agent 定义

**位置**：`packages/opencode/src/agent/agent.ts:178-187`

```typescript
title: {
  name: "title",
  mode: "primary",
  options: {},
  native: true,
  hidden: true,           // 隐藏 agent，用户不可见
  permission: agentPermission,
  prompt: PROMPT_TITLE,   // 使用 title.txt 作为系统提示词
  tools: {},              // 不使用任何工具
}
```

特点：

- **隐藏式服务**：`hidden: true` - 不向用户展示，后台自动执行
- **无工具依赖**：`tools: {}` - 纯文本生成任务
- **原生支持**：`native: true` - 内置的核心功能

---

## 两种使用场景

### 场景一：生成会话标题（Session Title）

**位置**：`packages/opencode/src/session/prompt.ts:1388-1455`

#### 触发时机

- 在对话的第一步时自动触发（`step === 1`，第 277-284 行）
- 仅在以下所有条件满足时执行：
  1. 不是子会话（`parentID` 不存在）
  2. 当前会话标题是默认标题（`isDefaultTitle()` 返回 `true`）
  3. 这是第一个真实用户消息（排除 synthetic 消息）

#### 执行流程

```typescript
// 1. 获取 title agent
const agent = await Agent.get("title")

// 2. 调用 LLM 生成标题
const result = await LLM.stream({
  agent,
  user: input.message.info as MessageV2.User,
  system: [],
  small: true, // 使用小模型以节省成本
  tools: {},
  model: agent.model
    ? await Provider.getModel(agent.model.providerID, agent.model.modelID)
    : ((await Provider.getSmallModel(input.providerID)) ?? (await Provider.getModel(input.providerID, input.modelID))),
  abort: new AbortController().signal,
  sessionID: input.session.id,
  retries: 2, // 失败时重试 2 次
  messages: [
    {
      role: "user",
      content: "Generate a title for this conversation:\n",
    },
    ...MessageV2.toModelMessage([
      {
        info: {
          /* ... */
        },
        parts: input.message.parts,
      },
    ]),
  ],
})
```

#### 结果处理

```typescript
const text = await result.text.catch((err) => log.error("failed to generate title", { error: err }))

if (text) {
  // 1. 清理文本：移除 think 标签和空行
  const cleaned = text
    .replace(/[\s\S]*?<\/think>\s*/g, "")
    .split("\n")
    .map((line) => line.trim())
    .find((line) => line.length > 0)

  if (!cleaned) return

  // 2. 限制长度：超过 100 字符截断为 97 字符 + "..."
  const title = cleaned.length > 100 ? cleaned.substring(0, 97) + "..." : cleaned

  // 3. 更新会话标题
  draft.title = title
}
```

#### 默认标题机制

**位置**：`packages/opencode/src/session/index.ts:25-36`

```typescript
const parentTitlePrefix = "New session - "
const childTitlePrefix = "Child session - "

function createDefaultTitle(isChild = false) {
  return (isChild ? childTitlePrefix : parentTitlePrefix) + new Date().toISOString()
}

export function isDefaultTitle(title: string) {
  // 正则匹配默认标题格式
  return new RegExp(`^(${parentTitlePrefix}|${childTitlePrefix})\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$`).test(
    title,
  )
}
```

**示例**：

- 父会话默认标题：`New session - 2025-12-31T10:30:45.123Z`
- 子会话默认标题：`Child session - 2025-12-31T10:30:45.123Z`

---

### 场景二：生成消息标题（Message Title）

**位置**：`packages/opencode/src/session/summary.ts:64-111`

#### 触发时机

在对话的第一步（step === 1），作为消息摘要的一部分生成：

- 用户消息有文本部分（且不是 synthetic）
- 用户消息还没有标题时（`!userMsg.summary?.title`）

#### 执行流程

```typescript
async function summarizeMessage(input: { messageID: string; messages: MessageV2.WithParts[] }) {
  // ... 前置处理 ...

  const textPart = msgWithParts.parts.find((p) => p.type === "text" && !p.synthetic) as MessageV2.TextPart
  if (textPart && !userMsg.summary?.title) {
    // 1. 获取 title agent
    const agent = await Agent.get("title")

    // 2. 选择模型（优先使用小模型）
    const small =
      (await Provider.getSmallModel(assistantMsg.providerID)) ??
      (await Provider.getModel(assistantMsg.providerID, assistantMsg.modelID))

    // 3. 调用 LLM 生成标题
    const stream = await LLM.stream({
      agent,
      user: userMsg,
      tools: {},
      model: agent.model ? await Provider.getModel(agent.model.providerID, agent.model.modelID) : small,
      small: true,
      messages: [
        {
          role: "user" as const,
          content: `
            The following is the text to summarize:
            <text>
            ${textPart?.text ?? ""}
            </text>
          `,
        },
      ],
      abort: new AbortController().signal,
      sessionID: userMsg.sessionID,
      system: [],
      retries: 3, // 失败时重试 3 次
    })

    // 4. 保存结果
    const result = await stream.text
    log.info("title", { title: result })
    userMsg.summary.title = result
    await Session.updateMessage(userMsg)
  }

  // ... 后续处理（生成 body 摘要）...
}
```

#### 使用场景

- 帮助用户在历史记录中快速定位特定消息
- 与会话标题配合，提供两层检索维度

---

## 两种场景的区别

| 特性         | 会话标题                     | 消息标题                       |
| ------------ | ---------------------------- | ------------------------------ |
| **触发时机** | 对话第一步（step === 1）     | 对话结束后的摘要生成           |
| **输入内容** | 整个第一条用户消息           | 用户消息的文本部分（textPart） |
| **更新位置** | `session.title`              | `userMsg.summary.title`        |
| **重试次数** | 2 次                         | 3 次                           |
| **模型选择** | 可配置 agent.model 或小模型  | 可配置 agent.model 或小模型    |
| **生成条件** | 仅第一次、非子会话、默认标题 | 用户消息尚未有标题             |

---

## 设计特点

### 1. 成本优化

- **小模型优先**：`small: true` 参数指示使用成本更低的模型
- **灵活模型选择**：agent 可配置专用模型，或使用提供商的小模型/主模型

### 2. 错误处理

- **重试机制**：会话标题 2 次，消息标题 3 次
- **静默失败**：捕获错误并记录日志，不阻塞主流程

### 3. 鲁棒性

- **输入清理**：自动移除 `think` 标签内容
- **长度控制**：自动截断超长标题
- **默认值保护**：始终输出有意义的标题，拒绝输入的情况在提示词中明确禁止

### 4. 用户体验

- **自动触发**：用户无需干预，系统自动完成
- **非阻塞**：异步执行，不影响对话主流程
- **智能判断**：避免重复生成，只在必要时执行

---

## 相关文件清单

- **提示词文件**：`packages/opencode/src/agent/prompt/title.txt`
- **Agent 定义**：`packages/opencode/src/agent/agent.ts:178-187`
- **会话标题生成**：`packages/opencode/src/session/prompt.ts:1388-1455`
- **消息标题生成**：`packages/opencode/src/session/summary.ts:64-111`
- **默认标题逻辑**：`packages/opencode/src/session/index.ts:25-36`
