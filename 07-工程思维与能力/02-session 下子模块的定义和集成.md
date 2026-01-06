# Session 子模块的定义和集成

## 1. 概述

Session 模块位于 `packages/opencode/src/session` 目录下，是一个用于管理 AI 对话会话的复杂系统。该模块采用了基于 **TypeScript 命名空间（Namespace）** 的模块化架构，将不同职责的功能分散到独立的子模块中。

## 2. 命名空间模块化架构

Session 模块的核心设计模式是：**每个文件导出一个命名空间，命名空间内包含相关的类型、常量、函数和事件定义**。

### 2.1 子模块列表

| 文件 | 命名空间 | 职责 |
|------|----------|------|
| `index.ts` | `Session` | 核心 CRUD 操作、会话生命周期管理 |
| `message-v2.ts` | `MessageV2` | 现代消息/Part 模式定义 |
| `message.ts` | `Message` | 遗留消息类型（已废弃） |
| `prompt.ts` | `SessionPrompt` | 用户提示词、命令处理、主循环 |
| `processor.ts` | `SessionProcessor` | LLM 流处理编排 |
| `llm.ts` | `LLM` | LLM 提供商抽象和流式传输 |
| `system.ts` | `SystemPrompt` | 系统提示词构建 |
| `compaction.ts` | `SessionCompaction` | 上下文窗口管理 |
| `retry.ts` | `SessionRetry` | 带指数退避的重试逻辑 |
| `status.ts` | `SessionStatus` | 会话状态（idle/busy/retry） |
| `summary.ts` | `SessionSummary` | 会话摘要和 diff 追踪 |
| `revert.ts` | `SessionRevert` | 基于快照的撤销功能 |
| `todo.ts` | `Todo` | 任务列表管理 |

## 3. 子模块定义模式

### 3.1 基本命名空间结构

每个子模块遵循统一的结构模式：

```typescript
// packages/opencode/src/session/index.ts
export namespace Session {
  // 1. 常量定义
  const log = Log.create({ service: "session" })
  const parentTitlePrefix = "New session - "

  // 2. Schema 定义（使用 Zod）
  export const Info = z.object({
    id: Identifier.schema("session"),
    projectID: z.string(),
    directory: z.string(),
    // ...
  })
  export type Info = z.output<typeof Info>

  // 3. 事件定义
  export const Event = {
    Created: BusEvent.define("session.created", z.object({ info: Info })),
    Updated: BusEvent.define("session.updated", z.object({ info: Info })),
    Deleted: BusEvent.define("session.deleted", z.object({ info: Info })),
    // ...
  }

  // 4. 函数定义（使用 fn() 包装器进行 Zod 验证）
  export const create = fn(
    z.object({
      parentID: Identifier.schema("session").optional(),
      title: z.string().optional(),
    }).optional(),
    async (input) => {
      // 实现逻辑
    }
  )

  // 5. 辅助函数
  export async function update(id: string, editor: (session: Info) => void) {
    // 实现
  }

  // 6. 自定义错误类
  export class BusyError extends Error {
    constructor(public readonly sessionID: string) {
      super(`Session ${sessionID} is busy`)
    }
  }
}
```

### 3.2 Schema-First 设计

所有数据结构都使用 **Zod** 进行定义，确保类型安全和运行时验证：

```typescript
// packages/opencode/src/session/message-v2.ts
export namespace MessageV2 {
  // Part 基础结构
  const PartBase = z.object({
    id: z.string(),
    sessionID: z.string(),
    messageID: z.string(),
  })

  // 具体 Part 类型
  export const TextPart = PartBase.extend({
    type: z.literal("text"),
    text: z.string(),
    synthetic: z.boolean().optional(),
    ignored: z.boolean().optional(),
    time: z.object({
      start: z.number(),
      end: z.number().optional(),
    }).optional(),
    metadata: z.record(z.string(), z.any()).optional(),
  })

  // 联合类型
  export const Part = z.discriminatedUnion("type", [
    TextPart,
    SubtaskPart,
    ReasoningPart,
    ToolPart,
    FilePart,
    StepStartPart,
    StepFinishPart,
    SnapshotPart,
    PatchPart,
    AgentPart,
    RetryPart,
    CompactionPart,
  ])
}
```

### 3.3 fn() 函数包装器

使用 `fn()` 工具函数包装所有公开函数，自动处理 Zod 验证：

```typescript
import { fn } from "@/util/fn"

export const create = fn(
  // 输入验证 schema
  z.object({
    parentID: Identifier.schema("session").optional(),
    title: z.string().optional(),
  }).optional(),
  // 实现函数（输入已验证）
  async (input) => {
    return createNext({
      parentID: input?.parentID,
      directory: Instance.directory,
      title: input?.title,
    })
  }
)
```

## 4. 子模块集成方式

### 4.1 直接导入

子模块之间通过直接 ES6 import 进行集成：

```typescript
// packages/opencode/src/session/processor.ts
import { MessageV2 } from "./message-v2"
import { Session } from "."
import { SessionSummary } from "./summary"
import { SessionRetry } from "./retry"
import { SessionStatus } from "./status"
import { SessionCompaction } from "./compaction"

export namespace SessionProcessor {
  export function create(input: {
    assistantMessage: MessageV2.Assistant
    sessionID: string
    model: Provider.Model
    abort: AbortSignal
  }) {
    // 使用其他子模块
    SessionStatus.set(input.sessionID, { type: "busy" })
    // ...
  }
}
```

### 4.2 事件驱动通信

通过中央事件总线（`Bus`）进行松耦合通信：

```typescript
// 发布事件
import { Bus } from "@/bus"

Bus.publish(Session.Event.Created, { info: result })
Bus.publish(MessageV2.Event.Updated, { info: msg })
Bus.publish(Session.Event.Error, { sessionID, error })

// 订阅事件（在其他模块中）
Bus.subscribe(Session.Event.Updated, (data) => {
  // 处理更新
})
```

### 4.3 存储抽象层

所有持久化通过统一的 Storage API 进行：

```typescript
import { Storage } from "../storage/storage"

// 写入
await Storage.write(["session", projectID, sessionID], data)

// 读取
const session = await Storage.read<Session.Info>(["session", projectID, sessionID])

// 列出
for (const item of await Storage.list(["session", projectID])) {
  const session = await Storage.read<Session.Info>(item)
}

// 更新
await Storage.update<Session.Info>(["session", projectID, sessionID], (draft) => {
  draft.title = "new title"
})

// 删除
await Storage.remove(["session", projectID, sessionID])
```

### 4.4 依赖关系图

```
Session (index.ts)
├── MessageV2 (message-v2.ts)
│   └── Message (message.ts) [遗留]
├── SessionPrompt (prompt.ts)
│   ├── LLM (llm.ts)
│   ├── SessionProcessor (processor.ts)
│   │   ├── MessageV2
│   │   ├── SessionStatus (status.ts)
│   │   ├── SessionRetry (retry.ts)
│   │   ├── SessionSummary (summary.ts)
│   │   └── SessionCompaction (compaction.ts)
│   ├── SystemPrompt (system.ts)
│   ├── SessionCompaction (compaction.ts)
│   ├── SessionSummary (summary.ts)
│   └── SessionRevert (revert.ts)
└── SessionRetry (retry.ts)
```

## 5. 核心数据结构

### 5.1 Session.Info（会话信息）

```typescript
{
  id: string                    // 唯一会话 ID
  projectID: string             // 项目 ID
  directory: string             // 工作目录
  parentID?: string             // 父会话 ID（用于 fork）
  title: string                 // 会话标题
  version: string               // 版本号
  time: {
    created: number             // 创建时间戳
    updated: number             // 更新时间戳
    compacting?: number         // 压缩中时间戳
    archived?: number           // 归档时间戳
  }
  summary?: {                   // 会话摘要
    additions: number           // 新增行数
    deletions: number           // 删除行数
    files: number               // 影响文件数
    diffs?: FileDiff[]          // 文件差异列表
  }
  share?: { url: string }       // 分享信息
  permission?: Ruleset          // 权限规则
  revert?: {                    // 撤销信息
    messageID: string
    partID?: string
    snapshot?: string
    diff?: string
  }
}
```

### 5.2 MessageV2.Info（消息信息）

两种消息类型：

**用户消息（User）:**
```typescript
{
  id: string
  sessionID: string
  role: "user"
  time: { created: number }
  agent: string                 // 代理名称
  model: {
    providerID: string
    modelID: string
  }
  system?: string               // 系统提示词
  tools?: Record<string, boolean>  // 可用工具
  summary?: {                   // 会话摘要
    title?: string
    body?: string
    diffs: FileDiff[]
  }
}
```

**助手消息（Assistant）:**
```typescript
{
  id: string
  sessionID: string
  role: "assistant"
  parentID: string              // 父消息 ID（用户消息）
  modelID: string
  providerID: string
  agent: string
  path: { cwd: string, root: string }
  time: {
    created: number
    completed?: number
  }
  cost: number                  // 成本
  tokens: {                     // Token 使用
    input: number
    output: number
    reasoning: number
    cache: { read: number, write: number }
  }
  finish?: string               // 完成原因
  error?: ErrorType             // 错误信息
}
```

### 5.3 MessageV2.Part（消息 Part）

12 种 Part类型：

| 类型 | 用途 |
|------|------|
| `TextPart` | 文本内容 |
| `SubtaskPart` | 子任务/代理调用 |
| `ReasoningPart` | 模型推理/思考过程 |
| `ToolPart` | 工具调用（pending/running/completed/error） |
| `FilePart` | 文件附件 |
| `StepStartPart` | 步骤开始标记 |
| `StepFinishPart` | 步骤完成标记 |
| `SnapshotPart` | 代码快照 |
| `PatchPart` | 差异补丁 |
| `AgentPart` | 代理引用 |
| `RetryPart` | 重试信息 |
| `CompactionPart` | 上下文压缩标记 |

## 6. 设计模式

### 6.1 命名空间模式

将相关功能组织到同一个命名空间下，提供清晰的逻辑分组。

### 6.2 事件驱动架构

通过中央事件总线实现模块间松耦合通信。

### 6.3 Schema-First

使用 Zod 进行类型定义和验证，确保类型安全。

### 6.4 函数式组合

使用小型、可组合的函数构建复杂功能。

### 6.5 策略模式

针对不同提供商使用不同的提示词模板（`prompt/anthropic.txt`、`prompt/gemini.txt` 等）。

### 6.6 状态机

会话状态（idle/busy/retry）、工具状态（pending/running/completed/error）。

### 6.7 重试与指数退避

通过 `SessionRetry` 实现弹性错误处理。

## 7. 关键工作流

### 7.1 主循环（SessionPrompt.loop）

```typescript
1. 获取会话消息
2. 处理待处理的压缩或子任务
3. 检查上下文溢出
4. 创建助手消息和处理器
5. 从注册表和 MCP 解析工具
6. 通过处理器流式传输 LLM 响应
7. 错误时处理重试
8. 返回完成的消息
```

### 7.2 处理器工作流（SessionProcessor.create）

```typescript
创建处理器 {
  初始化状态变量
  返回 {
    process(streamInput) {
      while (true) {
        获取 LLM 流
        for each (流事件) {
          switch (事件类型) {
            case "start": 设置会话忙碌
            case "reasoning-*": 处理模型思考
            case "text-*": 流式输出文本
            case "tool-*": 执行工具（带权限检查）
            case "start-step"/"finish-step": 跟踪边界
            case "error": 处理并重试错误
          }
          如果需要压缩则中断
        }
        捕获错误并决定是否重试
        如果无错误则返回
      }
    }
  }
}
```

### 7.3 压缩工作流（SessionCompaction）

- **Prune**: 清理旧的工具输出（>20k tokens）
- **Process**: 当上下文溢出时生成摘要
- 保持对话连续性

## 8. 总结

Session 模块的子模块化设计具有以下特点：

1. **清晰的组织结构**：每个文件一个命名空间，职责单一
2. **类型安全**：使用 Zod 进行 Schema 定义和运行时验证
3. **松耦合**：通过事件总线进行模块间通信
4. **可测试性**：函数式设计，易于单元测试
5. **可扩展性**：通过插件系统和 MCP 集成扩展功能
6. **弹性设计**：重试机制、状态管理、快照恢复

这种架构使得 Session 模块能够处理复杂的 AI 对话场景，同时保持代码的可维护性和可扩展性。
