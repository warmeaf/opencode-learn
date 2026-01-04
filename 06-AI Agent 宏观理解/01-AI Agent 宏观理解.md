# AI Agent 宏观理解

## 一、AI Agent 的定义

AI Agent（智能体）是为了实现某个目标，**循环调用工具的大语言模型**。

这个定义包含三个核心要素：

1. **大语言模型**：提供智能推理和决策能力
2. **工具**：提供与外部世界交互的能力
3. **循环调用**：不断感知、决策、执行、反馈的迭代过程

## 二、OpenCode 如何体现 AI Agent 定义

OpenCode 项目是一个完美的 AI Agent 实现，它通过以下代码架构体现了 Agent 的核心特征：

### 2.1 核心循环机制

#### 双层循环结构

OpenCode 实现了**双层循环**来驱动 Agent 的运行：

**外层循环** - `packages/opencode/src/session/prompt.ts:243`

```typescript
let step = 0
while (true) {
  SessionStatus.set(sessionID, { type: "busy" })
  log.info("loop", { step, sessionID })

  // 获取消息历史
  let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

  // 获取最后一条用户消息和助手消息
  // lastAssistant: AI Agent 对最后一条用户消息的完整响应（包含所有工具调用、推理、结果等）
  // 助手消息都是存储在数据库中的结构化数据，用于维护完整的对话历史，而不是大语言模型的临时输出。
  let lastUser: MessageV2.User | undefined
  let lastAssistant: MessageV2.Assistant | undefined

  // 准备调用 LLM
  const processor = SessionProcessor.create({ ... })
  const tools = await resolveTools({ agent, sessionID, model })

  // 处理当前步骤
  const result = await processor.process({
    user: lastUser,
    agent,
    abort,
    sessionID,
    messages: [...MessageV2.toModelMessage(sessionMessages)],
    tools,
    model,
  })

  step++
  if (result === "stop") break
  continue
}
```

这个外层循环体现了：

- **持续迭代**：不断从消息历史中提取最新的用户和助手消息
- **状态管理**：每一步都更新会话状态
- **步数追踪**：通过 `step` 变量跟踪执行步数
- **终止条件**：根据 `result` 决定是否继续循环

**内层循环** - `packages/opencode/src/session/processor.ts:45`

```typescript
async process(streamInput: LLM.StreamInput) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  let blocked = false
  let attempt = 0

  while (true) {
    try {
      let currentText: MessageV2.TextPart | undefined
      let reasoningMap: Record<string, MessageV2.ReasoningPart> = {}

      // 调用 LLM 流式接口
      const stream = await LLM.stream(streamInput)

      // 处理 LLM 的流式输出
      for await (const value of stream.fullStream) {
        input.abort.throwIfAborted()

        switch (value.type) {
          case "start":
            SessionStatus.set(input.sessionID, { type: "busy" })
            break

          case "reasoning-delta":
            // 处理推理过程
            part.text += value.text
            await Session.updatePart({ part, delta: value.text })
            break

          case "tool-input-start":
            // 工具调用开始
            const part = await Session.updatePart({
              type: "tool",
              tool: value.toolName,
              callID: value.id,
              state: { status: "pending", input: {} },
            })
            toolcalls[value.id] = part as MessageV2.ToolPart
            break

          case "tool-call":
            // 执行工具
            const match = toolcalls[value.toolCallId]
            if (match) {
              await Session.updatePart({
                ...match,
                tool: value.toolName,
                state: {
                  status: "running",
                  input: value.input,
                  time: { start: Date.now() },
                },
              })
            }
            break

          case "tool-result":
            // 工具执行结果
            const match = toolcalls[value.toolCallId]
            if (match) {
              await Session.updatePart({
                ...match,
                state: {
                  status: "completed",
                  input: value.input,
                  output: value.output.output,
                  title: value.output.title,
                  time: { start: match.state.time.start, end: Date.now() },
                },
              })
            }
            delete toolcalls[value.toolCallId]
            break

          case "finish":
            // 当前步骤完成
            break
        }
      }
    } catch (e) {
      // 错误处理和重试逻辑
      const retry = SessionRetry.retryable(error)
      if (retry !== undefined) {
        attempt++
        await SessionRetry.sleep(delay, input.abort)
        continue
      }
    }

    // 返回是否继续循环
    if (blocked) return "stop"
    if (input.assistantMessage.error) return "stop"
    return "continue"
  }
}
```

这个内层循环体现了：

- **流式处理**：逐个处理 LLM 返回的事件
- **工具生命周期**：pending → running → completed/error
- **错误恢复**：自动重试机制
- **循环控制**：根据状态决定继续或停止

### 2.2 工具系统

#### 工具定义

每个工具都遵循统一的定义规范 - `packages/opencode/src/tool/tool.ts`

```typescript
export interface Info<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
  id: string
  init: (ctx?: InitContext) => Promise<{
    description: string
    parameters: Parameters // Zod schema 用于参数验证
    execute(
      args: z.infer<Parameters>,
      ctx: Context,
    ): Promise<{
      title: string
      metadata: M
      output: string
      attachments?: MessageV2.FilePart[]
    }>
  }>
}
```

**示例：Read 工具** - `packages/opencode/src/tool/read.ts`

```typescript
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number().describe("The line number to start reading from (0-based)").optional(),
    limit: z.coerce.number().describe("The number of lines to read (defaults to 2000)").optional(),
  }),
  async execute(params, ctx) {
    const file = Bun.file(filepath)
    if (!(await file.exists())) {
      throw new Error(`File not found: ${filepath}`)
    }

    const lines = await file.text().then((text) => text.split("\n"))
    const content = lines.slice(offset, offset + limit).map(...)

    return {
      title: path.relative(Instance.worktree, filepath),
      output: `<file>\n${content.join("\n")}\n</file>`,
      metadata: { preview: raw.slice(0, 20).join("\n") },
    }
  },
})
```

**示例：Edit 工具** - `packages/opencode/src/tool/edit.ts`

```typescript
export const EditTool = Tool.define("edit", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string().describe("The absolute path to the file to modify"),
    oldString: z.string().describe("The text to replace"),
    newString: z.string().describe("The text to replace it with"),
    replaceAll: z.boolean().optional().describe("Replace all occurrences"),
  }),
  async execute(params, ctx) {
    const file = Bun.file(filePath)
    const contentOld = await file.text()
    const contentNew = replace(contentOld, params.oldString, params.newString, params.replaceAll)

    await file.write(contentNew)

    const diff = trimDiff(createTwoFilesPatch(filePath, filePath, contentOld, contentNew))

    return {
      title: path.relative(Instance.worktree, filePath),
      output: "",
      metadata: { diff, filediff },
    }
  },
})
```

#### 工具注册

**工具注册表** - `packages/opencode/src/tool/registry.ts`

```typescript
export namespace ToolRegistry {
  export const state = Instance.state(async () => {
    const custom = [] as Tool.Info[]

    // 加载自定义工具
    for await (const match of glob.scan({ cwd: dir })) {
      const mod = await import(match)
      for (const [id, def] of Object.entries<ToolDefinition>(mod)) {
        custom.push(fromPlugin(id, def))
      }
    }

    // 加载插件工具
    const plugins = await Plugin.list()
    for (const plugin of plugins) {
      for (const [id, def] of Object.entries(plugin.tool ?? {})) {
        custom.push(fromPlugin(id, def))
      }
    }

    return { custom }
  })

  export async function tools(providerID: string, agent?: Agent.Info) {
    const custom = await state().then((x) => x.custom)
    const config = await Config.get()

    return [
      InvalidTool,
      BashTool,
      ReadTool,
      GlobTool,
      GrepTool,
      EditTool,
      WriteTool,
      TaskTool,
      WebFetchTool,
      TodoWriteTool,
      TodoReadTool,
      WebSearchTool,
      CodeSearchTool,
      SkillTool,
      ...custom,
    ].map(async (t) => {
      return {
        id: t.id,
        ...(await t.init({ agent })),
      }
    })
  }
}
```

#### 可用工具列表

OpenCode 内置了丰富的工具集：

| 工具                   | 功能                 | 文件位置                                   |
| ---------------------- | -------------------- | ------------------------------------------ |
| **read**               | 读取文件内容         | `packages/opencode/src/tool/read.ts`       |
| **write**              | 写入新文件           | `packages/opencode/src/tool/write.ts`      |
| **edit**               | 编辑文件（替换文本） | `packages/opencode/src/tool/edit.ts`       |
| **bash**               | 执行 shell 命令      | `packages/opencode/src/tool/bash.ts`       |
| **glob**               | 文件模式匹配搜索     | `packages/opencode/src/tool/glob.ts`       |
| **grep**               | 文件内容正则搜索     | `packages/opencode/src/tool/grep.ts`       |
| **webfetch**           | 获取网页内容         | `packages/opencode/src/tool/webfetch.ts`   |
| **websearch**          | 网络搜索             | `packages/opencode/src/tool/websearch.ts`  |
| **codesearch**         | 代码搜索             | `packages/opencode/src/tool/codesearch.ts` |
| **task**               | 调用子 Agent         | `packages/opencode/src/tool/task.ts`       |
| **todowrite/todoread** | 任务管理             | `packages/opencode/src/tool/todo.ts`       |
| **skill**              | 执行自定义技能       | `packages/opencode/src/tool/skill.ts`      |

### 2.3 Agent 配置

#### Agent 定义结构 - `packages/opencode/src/agent/agent.ts:18`

```typescript
export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string().optional(),
    mode: z.enum(["subagent", "primary", "all"]),
    native: z.boolean().optional(),
    hidden: z.boolean().optional(),
    default: z.boolean().optional(),
    topP: z.number().optional(),
    temperature: z.number().optional(),
    color: z.string().optional(),
    permission: z.object({
      edit: Config.Permission, // 编辑权限
      bash: z.record(z.string(), Config.Permission), // bash 命令权限
      skill: z.record(z.string(), Config.Permission), // 技能权限
      webfetch: Config.Permission.optional(),
      doom_loop: Config.Permission.optional(),
      external_directory: Config.Permission.optional(),
    }),
    model: z
      .object({
        modelID: z.string(),
        providerID: z.string(),
      })
      .optional(),
    prompt: z.string().optional(),
    tools: z.record(z.string(), z.boolean()), // 工具启用/禁用
    options: z.record(z.string(), z.any()),
    maxSteps: z.number().int().positive().optional(), // 最大步数
  })
}
```

#### 内置 Agent 类型

**1. build Agent** - `packages/opencode/src/agent/agent.ts:118`

```typescript
build: {
  name: "build",
  tools: { ...defaultTools },
  options: {},
  permission: agentPermission,
  mode: "primary",
  native: true,
}
```

- **模式**：primary（主要 Agent）
- **权限**：完整的编辑和 bash 权限
- **用途**：通用的代码构建和开发任务

**2. plan Agent** - `packages/opencode/src/agent/agent.ts:126`

```typescript
plan: {
  name: "plan",
  options: {},
  permission: planPermission,  // 只读权限
  tools: { ...defaultTools },
  mode: "primary",
  native: true,
}
```

- **模式**：primary
- **权限**：受限权限（只读，有限的 bash）
- **用途**：规划和探索，不修改文件

**3. explore Agent** - `packages/opencode/src/agent/agent.ts:150`

```typescript
explore: {
  name: "explore",
  tools: {
    todowrite: false,
    todoread: false,
    edit: false,
    write: false,  // 禁用编辑工具
    ...defaultTools,
  },
  description: `Fast agent specialized for exploring codebases...`,
  prompt: PROMPT_EXPLORE,
  options: {},
  permission: agentPermission,
  mode: "subagent",
  native: true,
}
```

- **模式**：subagent（子 Agent）
- **权限**：只读权限
- **专用提示词**：探索代码库的系统提示
- **用途**：快速搜索和探索代码库

**4. general Agent** - `packages/opencode/src/agent/agent.ts:136`

```typescript
general: {
  name: "general",
  description: `General-purpose agent for researching complex questions and executing multi-step tasks.`,
  tools: {
    todowrite: false,
    todoread: false,
    ...defaultTools,
  },
  options: {},
  permission: agentPermission,
  mode: "subagent",
  native: true,
  hidden: true,
}
```

- **模式**：subagent
- **用途**：通过 Task 工具调用的通用子 Agent

### 2.4 LLM 调用与工具集成

#### LLM 流式接口 - `packages/opencode/src/session/llm.ts:43`

```typescript
export async function stream(input: StreamInput) {
  // 构建系统提示
  const system = SystemPrompt.header(input.model.providerID)
  system.push(
    [
      ...(input.agent.prompt ? [input.agent.prompt] : SystemPrompt.provider(input.model)),
      ...input.system,
      ...(input.user.system ? [input.user.system] : []),
    ]
    .filter((x) => x)
    .join("\n"),
  )

  // 调用本地resolveTools获取工具定义
  const tools = await resolveTools(input)

  // 调用 AI SDK 的 streamText
  return streamText({
    tools,                                  // 传递工具定义给 LLM
    maxOutputTokens,
    abortSignal: input.abort,
    messages: [
      ...system.map(x => ({ role: "system", content: x })),
      ...input.messages,
    ],
    model: wrapLanguageModel({ model: language, middleware: [...] }),
  })
}
```

#### 工具解析 - `packages/opencode/src/session/prompt.ts:573`

```typescript
async function resolveTools(input: {
  agent: Agent.Info
  model: Provider.Model
  sessionID: string
  tools?: Record<string, boolean>
}) {
  const tools: Record<string, AITool> = {}

  // 合并工具启用配置
  const enabledTools = pipe(
    input.agent.tools,
    mergeDeep(await ToolRegistry.enabled(input.agent)),
    mergeDeep(input.tools ?? {}),
  )

  // 加载并包装工具
  for (const item of await ToolRegistry.tools(input.model.providerID, input.agent)) {
    if (Wildcard.all(item.id, enabledTools) === false) continue

    tools[item.id] = tool({
      id: item.id,
      description: item.description,
      inputSchema: jsonSchema(item.parameters),
      async execute(args, options) {
        // 执行前插件钩子
        await Plugin.trigger("tool.execute.before", { ... }, { args })

        // 执行工具
        const result = await item.execute(args, {
          sessionID: input.sessionID,
          abort: options.abortSignal,
          messageID: input.processor.message.id,
          callID: options.toolCallId,
          extra: { model: input.model },
          agent: input.agent.name,
          metadata: async (val) => { /* 更新工具元数据 */ },
        })

        // 执行后插件钩子
        await Plugin.trigger("tool.execute.after", { ... }, result)

        return result
      },
    })
  }

  return tools
}
```

### 2.5 完整的 Agent 执行流程

#### 时序图

```
用户输入 (User Message)
    ↓
创建 User Message，保存到数据库
    ↓
进入外层循环 (while true)
    ↓
提取消息历史和最后一条消息
    ↓
创建 SessionProcessor
    ↓
解析可用的工具集合
    ↓
进入内层循环 (while true)
    ↓
调用 LLM.stream()
    ↓
LLM 开始流式输出
    ↓
┌────────────────────────────────────────┐
│ 处理流式事件：              │
│ 1. reasoning-delta (推理过程)  │
│ 2. tool-input-start (工具调用开始)│
│ 3. tool-call (执行工具)       │
│ 4. tool-result (工具结果)      │
│ 5. text-delta (文本输出)       │
│ 6. finish-step (步骤完成)       │
└────────────────────────────────────────┘
    ↓
工具执行：
  - 从 Tool Registry 获取工具定义
  - 验证参数 (Zod schema)
  - 执行工具逻辑
  - 返回结果
    ↓
将工具结果保存到 Message Part
    ↓
LLM 继续处理，生成下一步决策
    ↓
内层循环继续，直到 LLM 完成
    ↓
返回 "continue" 或 "stop"
    ↓
外层循环继续或终止
    ↓
最终结果返回给用户
```

#### 代码示例：完整的循环执行 - `packages/opencode/src/session/prompt.ts:243-672`

```typescript
export const loop = fn(Identifier.schema("session"), async (sessionID) => {
  const abort = start(sessionID)
  using _ = defer(() => cancel(sessionID))

  let step = 0
  while (true) {
    SessionStatus.set(sessionID, { type: "busy" })
    log.info("loop", { step, sessionID })

    // 获取消息历史
    let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

    // 查找最后一条用户和助手消息
    let lastUser: MessageV2.User | undefined
    let lastAssistant: MessageV2.Assistant | undefined

    for (let i = msgs.length - 1; i >= 0; i--) {
      const msg = msgs[i]
      if (!lastUser && msg.info.role === "user") lastUser = msg.info as MessageV2.User
      if (!lastAssistant && msg.info.role === "assistant") lastAssistant = msg.info as MessageV2.Assistant
      if (lastUser && lastAssistant) break
    }

    // 检查是否应该退出
    if (lastAssistant?.finish && lastUser.id < lastAssistant.id) {
      log.info("exiting loop", { sessionID })
      break
    }

    step++

    // 获取 Agent 和 Model
    const agent = await Agent.get(lastUser.agent)
    const model = await Provider.getModel(lastUser.model.providerID, lastUser.model.modelID)

    // 创建 SessionProcessor
    const processor = SessionProcessor.create({
      assistantMessage: (await Session.updateMessage({
        id: Identifier.ascending("message"),
        parentID: lastUser.id,
        role: "assistant",
        mode: agent.name,
        sessionID,
        ...
      })) as MessageV2.Assistant,
      sessionID,
      model,
      abort,
    })

    // 解析工具
    const tools = await resolveTools({
      agent,
      sessionID,
      model,
      tools: lastUser.tools,
      processor,
    })

    // 执行处理器
    const result = await processor.process({
      user: lastUser,
      agent,
      abort,
      sessionID,
      system: [...(await SystemPrompt.environment()), ...(await SystemPrompt.custom())],
      messages: [
        ...MessageV2.toModelMessage(msgs),
      ],
      tools,
      model,
    })

    // 决定是否继续
    if (result === "stop") break
    continue
  }

  // 返回最后的助手消息
  for await (const item of MessageV2.stream(sessionID)) {
    if (item.info.role === "assistant") return item
  }
})
```

### 2.6 智能特性

#### 1. Doom Loop 检测 - `packages/opencode/src/session/processor.ts:140`

```typescript
const lastThree = parts.slice(-DOOM_LOOP_THRESHOLD)

if (
  lastThree.length === DOOM_LOOP_THRESHOLD &&
  lastThree.every(
    (p) =>
      p.type === "tool" &&
      p.tool === value.toolName &&
      p.state.status !== "pending" &&
      JSON.stringify(p.state.input) === JSON.stringify(value.input),
  )
) {
  const permission = await Agent.get(input.assistantMessage.mode).then((x) => x.permission)
  if (permission.doom_loop === "ask") {
    await Permission.ask({
      type: "doom_loop",
      pattern: value.toolName,
      title: `Possible doom loop: "${value.toolName}" called ${DOOM_LOOP_THRESHOLD} times`,
    })
  } else if (permission.doom_loop === "deny") {
    throw new Permission.RejectedError(
      input.assistantMessage.sessionID,
      "doom_loop",
      value.toolCallId,
      `You seem to be stuck in a doom loop, please stop repeating same action`,
    )
  }
}
```

**防止无限循环**：检测连续 3 次使用相同工具和参数的调用。

#### 2. 权限控制 - `packages/opencode/src/tool/read.ts:33`

```typescript
if (!Filesystem.contains(Instance.directory, filepath)) {
  if (agent.permission.external_directory === "ask") {
    await Permission.ask({
      type: "external_directory",
      sessionID: ctx.sessionID,
      callID: ctx.callID,
      title: `Access file outside working directory: ${filepath}`,
    })
  } else if (agent.permission.external_directory === "deny") {
    throw new Permission.RejectedError(
      ctx.sessionID,
      "external_directory",
      ctx.callID,
      `File ${filepath} is not in the current working directory`,
    )
  }
}
```

**细粒度权限**：

- `edit`: allow/ask/deny
- `bash`: 按命令模式匹配
- `skill`: 按技能名称匹配
- `webfetch`: allow/ask/deny
- `external_directory`: allow/ask/deny
- `doom_loop`: ask/deny

#### 3. 重试机制 - `packages/opencode/src/session/processor.ts:349`

```typescript
const retry = SessionRetry.retryable(error)
if (retry !== undefined) {
  attempt++
  const delay = SessionRetry.delay(attempt, error.name === "APIError" ? error : undefined)
  SessionStatus.set(sessionID, {
    type: "retry",
    attempt,
    message: retry,
    next: Date.now() + delay,
  })
  await SessionRetry.sleep(delay, input.abort)
  continue
}
```

**自动重试**：对于可重试的错误（网络超时、速率限制等），自动重试并指数退避。

#### 4. 子 Agent 调用 - `packages/opencode/src/tool/task.ts`

```typescript
export const TaskTool = Tool.define("task", async () => {
  const agents = await Agent.list().then((x) => x.filter((a) => a.mode !== "primary"))
  const description = DESCRIPTION.replace(
    "{agents}",
    agents.map(a => `- ${a.name}: ${a.description}`).join("\n"),
  )

  return {
    parameters: z.object({
      description: z.string(),
      prompt: z.string(),
      subagent_type: z.string(),
      session_id: z.string().optional(),
    }),
    async execute(params, ctx) {
      const agent = await Agent.get(params.subagent_type)
      if (!agent) throw new Error(`Unknown agent type: ${params.subagent_type}`)

      // 创建新的会话用于子 Agent
      const session = await Session.create({
        parentID: ctx.sessionID,
        title: params.description + ` (@${agent.name} subagent)`,
      })

      // 调用子 Agent
      const result = await SessionPrompt.prompt({
        sessionID: session.id,
        model: { modelID, providerID },
        agent: agent.name,
        parts: [{ type: "text", text: params.prompt }],
      })

      return {
        title: params.description,
        metadata: { sessionId: session.id },
        output: text + `\n\nsession_id: ${session.id}`,
      }
    },
  }
}
```

**多 Agent 协作**：主 Agent 可以调用子 Agent 处理特定任务，实现分层处理。

#### 5. 上下文压缩 - `packages/opencode/src/session/prompt.ts:460`

```typescript
// 检查是否需要压缩
if (
  lastFinished &&
  lastFinished.summary !== true &&
  (await SessionCompaction.isOverflow({ tokens: lastFinished.tokens, model }))
) {
  await SessionCompaction.create({
    sessionID,
    agent: lastUser.agent,
    model: lastUser.model,
    auto: true,
  })
  continue
}
```

**智能上下文管理**：当 token 使用超过阈值时，自动压缩历史消息，保持上下文在合理范围。

## 三、OpenCode Agent 的核心特征总结

### 3.1 循环调用工具 ✅

**代码证据**：

- `SessionPrompt.loop()` - `packages/opencode/src/session/prompt.ts:243`
- `SessionProcessor.process()` - `packages/opencode/src/session/processor.ts:45`

**实现方式**：

```
while (true) {
  // 1. 感知：读取消息历史
  const msgs = await getMessages()

  // 2. 决策：调用 LLM
  const stream = await LLM.stream({ messages: msgs, tools })

  // 3. 执行：处理工具调用
  for await (const event of stream) {
    if (event.type === "tool-call") {
      const result = await executeTool(event.tool, event.args)
      // 将结果反馈给 LLM
    }
  }

  // 4. 反馈：继续循环
  continue
}
```

### 3.2 大语言模型 ✅

**代码证据**：

- `LLM.stream()` - `packages/opencode/src/session/llm.ts:43`

**模型支持**：

- 多个 Provider（OpenAI、Anthropic、Google、OpenCode 等）
- 统一接口封装
- 流式输出处理
- 工具调用支持

### 3.3 丰富的工具集 ✅

**代码证据**：

- `ToolRegistry` - `packages/opencode/src/tool/registry.ts`
- 15+ 内置工具
- 插件系统支持自定义工具

**工具能力**：

- 文件操作（read、write、edit）
- 代码搜索（glob、grep、codesearch）
- 命令执行（bash）
- 网络访问（webfetch、websearch）
- 任务管理（todowrite、todoread）
- Agent 调用（task）

### 3.4 智能决策 ✅

**特性**：

- Doom Loop 检测和防护
- 权限细粒度控制
- 自动重试和错误恢复
- 上下文智能压缩
- 多 Agent 协作

## 四、与传统 ChatBot 的区别

| 特性         | 传统 ChatBot       | OpenCode Agent      |
| ------------ | ------------------ | ------------------- |
| **工具调用** | 一次调用，单次响应 | 循环调用，多次交互  |
| **状态管理** | 无状态             | 维护完整的会话历史  |
| **自主性**   | 被动响应           | 主动决策和执行      |
| **错误处理** | 简单重试           | 智能重试 + 降级策略 |
| **任务分解** | 依赖用户指导       | 自动分解和执行      |
| **多 Agent** | 不支持             | 支持子 Agent 调用   |

## 五、总结

OpenCode 项目完美体现了 AI Agent 的核心定义：

1. **大语言模型**：通过 `LLM.stream()` 提供智能推理
2. **工具**：通过 `ToolRegistry` 管理丰富的工具集
3. **循环调用**：通过双层 `while (true)` 循环实现持续交互

**核心设计模式**：

- **感知-决策-执行-反馈循环**：外层循环控制整体流程
- **事件驱动工具执行**：内层循环处理流式输出
- **状态机设计**：工具状态（pending → running → completed/error）
- **插件化架构**：支持自定义工具和 Agent
- **安全机制**：权限控制、Doom Loop 检测、自动重试

这种架构使得 OpenCode 不仅仅是一个聊天机器人，而是一个能够理解用户目标、分解任务、调用工具、执行操作、验证结果的**自主智能体**。

---

**参考代码位置**：

- Agent 定义：`packages/opencode/src/agent/agent.ts`
- 工具定义：`packages/opencode/src/tool/`
- 循环逻辑：`packages/opencode/src/session/prompt.ts:231-523`
- 流式处理：`packages/opencode/src/session/processor.ts:42-408`
- LLM 调用：`packages/opencode/src/session/llm.ts:43-200`
