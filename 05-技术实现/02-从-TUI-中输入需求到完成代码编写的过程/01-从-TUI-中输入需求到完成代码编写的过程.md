# 从 TUI 中输入需求到完成代码编写的过程

## 概览

本文档详细描述用户在 OpenCode TUI（Terminal User Interface）中输入需求后，系统如何处理需求、调用 AI、执行工具、最终完成代码编写的完整流程。

## 架构分层

整个流程分为以下几个核心层次：

```
┌─────────────────────────────────────────────────────────────┐
│                     TUI 表现层                              │
│  (packages/opencode/src/cli/cmd/tui/)                      │
│  - app.tsx: 主应用                                         │
│  - routes/session/index.tsx: 会话页面                        │
│  - component/prompt: 输入组件                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     网络通信层                              │
│  (packages/opencode/src/server/)                           │
│  - server.ts: HTTP 服务器                                  │
│  - tui/submitPrompt: API 端点                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    会话处理层                               │
│  (packages/opencode/src/session/)                          │
│  - prompt.ts: SessionPrompt.prompt()                       │
│  - processor.ts: SessionProcessor.process()                  │
│  - message-v2.ts: 消息定义                                │
│  - index.ts: 会话管理                                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     AI 执行层                              │
│  (packages/opencode/src/)                                 │
│  - session/llm.ts: LLM 调用                               │
│  - agent/agent.ts: Agent 配置                              │
│  - tool/registry.ts: 工具注册表                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    工具执行层                              │
│  (packages/opencode/src/tool/)                             │
│  - edit.ts: 文件编辑                                      │
│  - write.ts: 文件写入                                     │
│  - bash.ts: 命令执行                                     │
│  - grep.ts: 代码搜索                                      │
│  - read.ts: 文件读取                                      │
│  - task.ts: 子任务执行                                     │
└─────────────────────────────────────────────────────────────┘
```

## 详细流程

### 阶段 1: TUI 输入

**位置**: `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`

1. **用户在输入框中输入需求**
   - Prompt 组件捕获用户输入
   - 支持 Markdown 语法和文件引用（如 `@file.ts`）
   - 支持多行输入

2. **输入验证和预处理**
   - 解析 Markdown 格式
   - 识别文件引用和命令
   - 将输入转换为标准格式

3. **提交到后端**
   ```typescript
   // 通过 SDK 调用 API
   sdk.client.session.prompt({
     sessionID,
     ...selectedModel,
     messageID,
     agent: local.agent.current().name,
     model: selectedModel,
     parts: [
       {
         id: Identifier.ascending("part"),
         type: "text",
         text: inputText,
       },
       ...nonTextParts.map((x) => ({
         id: Identifier.ascending("part"),
         ...x,
       })),
     ],
   })
   ```

**关键文件**:

- `packages/opencode/src/cli/cmd/tui/component/prompt/`: Prompt 组件实现
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:300+`: Session 页面逻辑

### 阶段 2: 网络传输

**位置**: `packages/opencode/src/server/server.ts:2390-2410`

1. **接收 TUI 提交请求**

   ```typescript
   .post(
     "/tui/submit-prompt",
     async (c) => {
       // 发布 TUI 命令执行事件
       await Bus.publish(TuiEvent.CommandExecute, {
         command: "prompt.submit",
       })
       return c.json(true)
     },
   )
   ```

2. **事件路由**
   - 通过 Bus 系统发布命令执行事件
   - TUI 通过监听事件触发相应命令处理

**关键文件**:

- `packages/opencode/src/server/server.ts:2390`: submitPrompt API
- `packages/opencode/src/session/prompt.ts:1282`: command() 函数

### 阶段 3: 会话初始化

**位置**: `packages/opencode/src/session/prompt.ts:140-152`

1. **创建用户消息**

   ```typescript
   const message = await createUserMessage(input)
   ```

2. **解析 Prompt Parts**
   - 文本部分 (TextPart)
   - 文件引用 (FilePart)
   - Agent 引用 (AgentPart)
   - 子任务 (SubtaskPart)

3. **存储消息和 Part**
   - 保存到 Storage
   - 发布事件到 Bus

**关键代码**: `packages/opencode/src/session/prompt.ts:717-1002` - `createUserMessage()`

### 阶段 4: 主循环启动

**位置**: `packages/opencode/src/session/prompt.ts:230-563`

`SessionPrompt.loop()` 是核心处理循环：

```typescript
export const loop = fn(Identifier.schema("session"), async (sessionID) => {
  while (true) {
    // 1. 设置会话状态为 busy
    SessionStatus.set(sessionID, { type: "busy" })

    // 2. 获取历史消息
    let msgs = await MessageV2.filterCompacted(MessageV2.stream(sessionID))

    // 3. 查找最后的用户消息和助手消息
    let lastUser: MessageV2.User | undefined
    let lastAssistant: MessageV2.Assistant | undefined
    let lastFinished: MessageV2.Assistant | undefined
    let tasks: (MessageV2.CompactionPart | MessageV2.SubtaskPart)[] = []
    for (let i = msgs.length - 1; i >= 0; i--) {
      const msg = msgs[i]
      if (!lastUser && msg.info.role === "user") lastUser = msg.info as MessageV2.User
      if (!lastAssistant && msg.info.role === "assistant") lastAssistant = msg.info as MessageV2.Assistant
      if (!lastFinished && msg.info.role === "assistant" && msg.info.finish)
        lastFinished = msg.info as MessageV2.Assistant
      if (lastUser && lastFinished) break
      const task = msg.parts.filter((part) => part.type === "compaction" || part.type === "subtask")
      if (task && !lastFinished) {
        tasks.push(...task)
      }
    }

    if (!lastUser) throw new Error("No user message found in stream. This should never happen.")

    // 4. 检查是否应该退出循环
    if (
      lastAssistant?.finish &&
      !["tool-calls", "unknown"].includes(lastAssistant.finish) &&
      lastUser.id < lastAssistant.id
    ) {
      break
    }

    step++

    // 5. 确保标题（如果是第一次）
    if (step === 1)
      ensureTitle({
        session: await Session.get(sessionID),
        modelID: lastUser.model.modelID,
        providerID: lastUser.model.providerID,
        message: msgs.find((m) => m.info.role === "user")!,
        history: msgs,
      })

    const model = await Provider.getModel(lastUser.model.providerID, lastUser.model.modelID)
    const task = tasks.pop()

    // 6. 处理待处理的 subtask
    if (task?.type === "subtask") {
      // 执行子任务：创建 assistant 消息，调用 TaskTool，更新状态
      // 完整实现包括创建 synthetic user message 防止某些模型错误
      continue
    }

    // 7. 处理待处理的 compaction
    if (task?.type === "compaction") {
      const result = await SessionCompaction.process({
        messages: msgs,
        parentID: lastUser.id,
        abort,
        sessionID,
        auto: task.auto,
      })
      if (result === "stop") break
      continue
    }

    // 8. 检查上下文溢出
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

    // 9. 正常处理
    const agent = await Agent.get(lastUser.agent)
    const maxSteps = agent.maxSteps ?? Infinity
    const isLastStep = step >= maxSteps

    msgs = insertReminders({
      messages: msgs,
      agent,
    })

    // 10. 创建 SessionProcessor
    const processor = SessionProcessor.create({
      assistantMessage: (await Session.updateMessage({
        id: Identifier.ascending("message"),
        parentID: lastUser.id,
        role: "assistant",
        mode: agent.name,
        agent: agent.name,
        path: {
          cwd: Instance.directory,
          root: Instance.worktree,
        },
        cost: 0,
        tokens: {
          input: 0,
          output: 0,
          reasoning: 0,
          cache: { read: 0, write: 0 },
        },
        modelID: model.id,
        providerID: model.providerID,
        time: {
          created: Date.now(),
        },
        sessionID,
      })) as MessageV2.Assistant,
      sessionID: sessionID,
      model,
      abort,
    })

    // 11. 解析可用工具
    const tools = await resolveTools({
      agent,
      sessionID,
      model,
      tools: lastUser.tools,
      processor,
    })

    // 12. 处理 LLM 响应
    const sessionMessages = clone(msgs)
    await Plugin.trigger("experimental.chat.messages.transform", {}, { messages: sessionMessages })

    const result = await processor.process({
      user: lastUser,
      agent,
      abort,
      sessionID,
      system: [...(await SystemPrompt.environment()), ...(await SystemPrompt.custom())],
      messages: [
        ...MessageV2.toModelMessage(sessionMessages),
        ...(isLastStep
          ? [
              {
                role: "assistant" as const,
                content: MAX_STEPS,
              },
            ]
          : []),
      ],
      tools,
      model,
    })

    if (result === "stop") break
    continue
  }
})
```

**注**: 为简化展示，子任务处理逻辑省略了详细实现，包括 TaskTool 初始化、执行、错误处理等。完整实现请参考源码 `packages/opencode/src/session/prompt.ts:290-443`。

### 阶段 5: AI 调用

**位置**: `packages/opencode/src/session/llm.ts:35-150`

```typescript
export async function stream(input: StreamInput) {
  // 1. 构建 System Prompt
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

  // 2. 获取模型和参数
  const provider = await Provider.getProvider(input.model.providerID)
  const params = {
    temperature: input.agent.temperature ?? ProviderTransform.temperature(input.model),
    topP: input.agent.topP ?? ProviderTransform.topP(input.model),
    options: {
      /* 合并各种选项 */
    },
  }

  // 3. 解析工具
  const tools = await resolveTools(input)

  // 4. 调用 AI SDK 的 streamText
  // 注：为简洁展示，省略了以下参数：topK, onError, experimental_repairToolCall,
  // providerOptions, activeTools, headers, maxRetries
  // 实际代码中 messages 参数和 model 参数的构建方式也略有不同
  // 完整实现请参考源码 packages/opencode/src/session/llm.ts:116-187
  return streamText({
    temperature: params.temperature,
    topP: params.topP,
    tools,
    maxOutputTokens,
    abortSignal: input.abort,
    model: language,
    messages: [{ role: "system", content: system.join("\n") }, ...input.messages],
  })
}
```

### 阶段 6: 流式响应处理

**位置**: `packages/opencode/src/session/processor.ts:42-342`

`SessionProcessor.process()` 处理 AI 返回的流式响应：

```typescript
async process(streamInput: LLM.StreamInput) {
  while (true) {
    try {
      const stream = await LLM.stream(streamInput)

      for await (const value of stream.fullStream) {
        input.abort.throwIfAborted()

        switch (value.type) {
          case "start":
            SessionStatus.set(input.sessionID, { type: "busy" })
            break

          case "reasoning-start":
            reasoningMap[value.id] = { /* 创建 ReasoningPart */ }
            break

          case "reasoning-delta":
            reasoningMap[value.id].text += value.text
            await Session.updatePart({ part, delta: value.text })
            break

          case "reasoning-end":
            await Session.updatePart(part)
            delete reasoningMap[value.id]
            break

          case "tool-input-start":
            const part = await Session.updatePart({
              id: toolcalls[value.id]?.id ?? Identifier.ascending("part"),
              type: "tool",
              tool: value.toolName,
              callID: value.id,
              state: { status: "pending", input: {} }
            })
            toolcalls[value.id] = part
            break

          case "tool-call":
            const match = toolcalls[value.toolCallId]
            const part = await Session.updatePart({
              ...match,
              tool: value.toolName,
              state: { status: "running", input: value.input }
            })
            break

          case "tool-result": {
            const match = toolcalls[value.toolCallId]
            if (match && match.state.status === "running") {
              await Session.updatePart({
                ...match,
                state: {
                  status: "completed",
                  input: value.input,
                  output: value.output.output,
                  metadata: value.output.metadata,
                  title: value.output.title,
                  time: {
                    start: match.state.time.start,
                    end: Date.now(),
                  },
                  attachments: value.output.attachments,
                },
              })

              delete toolcalls[value.toolCallId]
            }
            break
          }

          case "tool-error":
            await Session.updatePart({
              ...match,
              state: { status: "error", error: value.error.toString() }
            })
            break

          case "text-start":
            currentText = { type: "text", text: "" }
            break

          case "text-delta":
            currentText.text += value.text
            await Session.updatePart({ part: currentText, delta: value.text })
            break

          case "text-end":
            await Session.updatePart(currentText)
            currentText = undefined
            break

          case "start-step":
            snapshot = await Snapshot.track()
            await Session.updatePart({ type: "step-start", snapshot })
            break

          case "finish-step":
            await Session.updatePart({
              type: "step-finish",
              snapshot: await Snapshot.track(),
              cost: usage.cost,
              tokens: usage.tokens
            })
            if (snapshot) {
              const patch = await Snapshot.patch(snapshot)
              if (patch.files.length) {
                await Session.updatePart({
                  type: "patch",
                  hash: patch.hash,
                  files: patch.files
                })
              }
            }
            break

          case "error":
            throw value.error
        }
      }
    } catch (e: any) {
      const error = MessageV2.fromError(e, { providerID: input.model.providerID })
      const retry = SessionRetry.retryable(error)
      if (retry !== undefined) {
        attempt++
        const delay = SessionRetry.delay(attempt, error)
        SessionStatus.set(input.sessionID, {
          type: "retry",
          attempt,
          message: retry,
          next: Date.now() + delay
        })
        await SessionRetry.sleep(delay, input.abort)
        continue
      }
      input.assistantMessage.error = error
      break
    }
  }
  return "continue"
}
```

### 阶段 7: 工具执行

**位置**: `packages/opencode/src/session/prompt.ts:572-715`

工具通过 `resolveTools()` 解析和注册：

```typescript
async function resolveTools(input: {
  agent: Agent.Info
  model: Provider.Model
  sessionID: string
  tools?: Record<string, boolean>
  processor: SessionProcessor.Info
}) {
  const tools: Record<string, AITool> = {}

  const enabledTools = pipe(
    input.agent.tools,
    mergeDeep(await ToolRegistry.enabled(input.agent)),
    mergeDeep(input.tools ?? {})
  )

  for (const item of await ToolRegistry.tools(input.model.providerID, input.agent)) {
    if (Wildcard.all(item.id, enabledTools) === false) continue

    tools[item.id] = tool({
      id: item.id,
      description: item.description,
      // 注：实际代码中先计算 schema，然后传递给 jsonSchema
      // const schema = ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))
      // inputSchema: jsonSchema(schema as any)
      inputSchema: jsonSchema(ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))),
      async execute(args, options) {
        await Plugin.trigger("tool.execute.before", { tool: item.id, ... })

        const result = await item.execute(args, {
          sessionID: input.sessionID,
          abort: options.abortSignal!,
          messageID: input.processor.message.id,
          callID: options.toolCallId,
          extra: { model: input.model },
          agent: input.agent.name,
          metadata: async (val) => {
            // 注：实际代码中会先检查 tool part 的状态，然后再更新
            // const match = input.processor.partFromToolCall(options.toolCallId)
            // if (match && match.state.status === "running") {
            //   await Session.updatePart({ ... })
            // }
            await Session.updatePart({ /* 更新 part */ })
          }
        })

        await Plugin.trigger("tool.execute.after", { tool: item.id, ... })

        return result
      },
      toModelOutput(result) {
        return { type: "text", value: result.output }
      }
    })
  }

  for (const [key, item] of Object.entries(await MCP.tools())) {
    // 类似地包装 MCP 工具
  }

  return tools
}
```

### 阶段 8: 常用工具实现

#### Edit Tool - 文件编辑

**位置**: `packages/opencode/src/tool/edit.ts:27-100`

```typescript
export const EditTool = Tool.define("edit", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string(),
    oldString: z.string(),
    newString: z.string(),
    replaceAll: z.boolean().optional(),
  }),
  async execute(params, ctx) {
    if (agent.permission.edit === "ask") {
      await Permission.ask({
        type: "edit",
        sessionID: ctx.sessionID,
        title: "Edit this file: " + filePath,
        metadata: { filePath, diff },
      })
    }

    if (!Filesystem.contains(Instance.directory, filePath)) {
      if (agent.permission.external_directory === "ask") {
        await Permission.ask({
          /* ... */
        })
      }
    }

    await FileTime.withLock(filePath, async () => {
      if (params.oldString === "") {
        await Bun.write(filePath, params.newString)
      } else {
        let content = await Bun.file(filePath).text()
        content = content.replace(params.oldString, params.newString, params.replaceAll ? "g" : undefined)
        await Bun.write(filePath, content)
      }
    })

    await Bus.publish(File.Event.Edited, { file: filePath })
    contentNew = await file.text()
    diff = trimDiff(
      createTwoFilesPatch(filePath, filePath, normalizeLineEndings(contentOld), normalizeLineEndings(contentNew)),
    )
    FileTime.read(ctx.sessionID, filePath)
  })

  let output = ""
  const diagnostics = await LSP.diagnostics()
  const normalizedFilePath = Filesystem.normalizePath(filePath)
  const issues = diagnostics[normalizedFilePath] ?? []
  const errors = issues.filter((item) => item.severity === 1)
  if (errors.length > 0) {
    const limited = errors.slice(0, MAX_DIAGNOSTICS_PER_FILE)
    const suffix =
      errors.length > MAX_DIAGNOSTICS_PER_FILE ? `\n... and ${errors.length - MAX_DIAGNOSTICS_PER_FILE} more` : ""
    output += `\nThis file has errors, please fix\n<file_diagnostics>\n${limited.map(LSP.Diagnostic.pretty).join("\n")}${suffix}\n</file_diagnostics>\n`
  }

  const filediff: Snapshot.FileDiff = {
    file: filePath,
    before: contentOld,
    after: contentNew,
    additions: 0,
    deletions: 0,
  }
  for (const change of diffLines(contentOld, contentNew)) {
    if (change.added) filediff.additions += change.count || 0
    if (change.removed) filediff.deletions += change.count || 0
  }

  return {
    metadata: {
      diagnostics,
      diff,
      filediff,
    },
    title: `${path.relative(Instance.worktree, filePath)}`,
    output,
  }
},
})
```

### 阶段 9: 权限管理

**位置**: `packages/opencode/src/permission/`

权限系统控制工具的执行：

```typescript
export namespace Permission {
  export async function ask(input: {
    type: string
    title: string
    pattern?: string | string[]
    callID?: string
    sessionID: string
    messageID: string
    metadata: Record<string, any>
  }) {
    const { pending, approved } = state()
    const approvedForSession = approved[input.sessionID] || {}
    const keys = toKeys(input.pattern, input.type)
    if (covered(keys, approvedForSession)) return

    const info: Info = {
      id: Identifier.ascending("permission"),
      type: input.type,
      pattern: input.pattern,
      sessionID: input.sessionID,
      messageID: input.messageID,
      callID: input.callID,
      title: input.title,
      metadata: input.metadata,
      time: {
        created: Date.now(),
      },
    }

    switch (
      await Plugin.trigger("permission.ask", info, {
        status: "ask",
      }).then((x) => x.status)
    ) {
      case "deny":
        throw new RejectedError(info.sessionID, info.id, info.callID, info.metadata)
      case "allow":
        return
    }

    pending[input.sessionID] = pending[input.sessionID] || {}
    return new Promise<void>((resolve, reject) => {
      pending[input.sessionID][info.id] = {
        info,
        resolve,
        reject,
      }
      Bus.publish(Event.Updated, info)
    })
  }

  export class RejectedError extends Error {
    constructor(
      public readonly sessionID: string,
      public readonly permissionID: string,
      public readonly toolCallID?: string,
      public readonly metadata?: Record<string, any>,
      public readonly reason?: string,
    ) {
      super(
        reason !== undefined
          ? reason
          : `The user rejected permission to use this specific tool call. You may try again with different parameters.`,
      )
    }
  }
}
```

### 阶段 10: 快照和差异跟踪

**位置**: `packages/opencode/src/snapshot/`

```typescript
export namespace Snapshot {
  export async function track(): Promise<string> {
    // 使用 git 创建快照
    if (Instance.project.vcs !== "git") return
    const cfg = await Config.get()
    if (cfg.snapshot === false) return

    // 初始化 git（如需要）
    const git = gitdir()
    if (await fs.mkdir(git, { recursive: true })) {
      await $`git init`
        .env({
          ...process.env,
          GIT_DIR: git,
          GIT_WORK_TREE: Instance.worktree,
        })
        .quiet()
        .nothrow()
      await $`git --git-dir ${git} config core.autocrlf false`.quiet().nothrow()
    }

    // 添加所有文件到 git
    await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet().cwd(Instance.directory).nothrow()

    // 创建 git tree 并返回 hash
    const hash = await $`git --git-dir ${git} --work-tree ${Instance.worktree} write-tree`
      .quiet()
      .cwd(Instance.directory)
      .nothrow()
      .text()

    return hash.trim()
  }

  export async function patch(hash: string): Promise<Patch> {
    // 使用 git diff 获取变更文件
    const git = gitdir()
    await $`git --git-dir ${git} --work-tree ${Instance.worktree} add .`.quiet().cwd(Instance.directory).nothrow()

    const result =
      await $`git -c core.autocrlf=false --git-dir ${git} --work-tree ${Instance.worktree} diff --no-ext-diff --name-only ${hash} -- .`
        .quiet()
        .cwd(Instance.directory)
        .nothrow()

    // 处理失败情况
    if (result.exitCode !== 0) {
      return { hash, files: [] }
    }

    // 解析变更文件列表
    const files = result
      .text()
      .trim()
      .split("\n")
      .map((x) => x.trim())
      .filter(Boolean)
      .map((x) => path.join(Instance.worktree, x))

    return { hash, files }
  }
}
```

### 阶段 11: 回到 TUI 显示

**位置**: `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`

1. **通过 SSE 事件接收更新**

   ```typescript
   sdk.event.on(MessageV2.Event.PartUpdated.type, (evt) => {
     // 更新 UI 中的 part
   })
   ```

2. **渲染不同的 Part 类型**
   - TextPart: 显示文本
   - ReasoningPart: 显示推理过程（可折叠）
   - ToolPart: 显示工具调用（可展开查看详情）
   - FilePart: 显示文件内容
   - PatchPart: 显示代码差异

3. **实时更新**
   - 流式显示 AI 响应
   - 动态更新工具状态
   - 显示进度和错误

## 数据流图

```
用户输入 (TUI)
     │
     ▼
Prompt Component
     │
     ▼ POST /tui/submit-prompt
HTTP Server
     │
     ▼ Bus.publish(TuiEvent.CommandExecute)
     │   └─ command: "prompt.submit"
     │
     ▼ TUI 命令处理
     │   └─ SessionPrompt.prompt()
     │
     ▼ createUserMessage()
     │   ├─ Parse TextPart
     │   ├─ Parse FilePart (@file.ts)
     │   └─ Parse AgentPart (@agent)
     │
     ▼ Session.updateMessage()
     │   └─ Storage.write(["message", ...])
     │
     ▼ SessionPrompt.loop()
     │   ├─ Get history messages
     │   ├─ Check pending subtasks/compactions
     │   ├─ Check context overflow
     │   ├─ SessionProcessor.create()
     │   │
     │   ▼ processor.process()
     │       │
     │       ▼ LLM.stream()
     │           │
     │           ▼ streamText() [AI SDK]
     │               │
     │               ▼ Stream Events
     │                   ├─ reasoning-start
     │                   ├─ text-delta
     │                   ├─ tool-call
     │                   │   │
     │                   │   ▼ Tool.execute()
     │                   │       ├─ Permission.ask()
     │                   │       ├─ Execute (Edit/Write/Bash/etc.)
     │                   │       └─ Return result
     │                   │
     │                   ├─ tool-result
     │                   ├─ text-end
     │                   └─ finish-step
     │                       │
     │                       ▼ Snapshot.track() & Snapshot.patch()
     │                           └─ Git-based diff tracking
     │
     ▼ Storage updates (via Session.updatePart)
     │   ├─ TextPart
     │   ├─ ToolPart
     │   ├─ PatchPart
     │   └─ etc.
     │
     ▼ Bus.publish(Event.*)
     │
     ▼ SSE (Server-Sent Events)
     │
     ▼ TUI SDK
     │
     ▼ UI Updates
     │   ├─ Display text
     │   ├─ Show tool calls
     │   ├─ Show diffs
     │   └─ Show errors
```

## 关键设计模式

### 1. 事件驱动架构

使用 Bus 系统进行解耦通信：

```typescript
Bus.publish(Session.Event.Updated, { info: session })
Bus.on(Session.Event.Updated.type, (event) => {
  /* 处理更新 */
})
```

### 2. 流式处理

所有 AI 响应都是流式处理的：

```typescript
for await (const value of stream.fullStream) {
  switch (value.type) {
    case "text-delta":
      /* 实时显示文本 */ break
    case "tool-call":
      /* 执行工具 */ break
  }
}
```

### 3. 快照和回滚

使用 git 作为后端跟踪文件变化：

```typescript
const snapshot = await Snapshot.track()
// 使用 git write-tree 创建快照，返回 git tree hash
// 执行工具...
const patch = await Snapshot.patch(snapshot)
// 使用 git diff --name-only 获取变更文件列表
```

**注**: 快照系统基于 git 实现，利用 git 的 tree 对象和 diff 能力来跟踪文件变化，而不是使用自定义存储和哈希比较。

### 4. 权限检查

工具执行前进行权限验证：

```typescript
if (agent.permission.edit === "ask") {
  await Permission.ask({
    /* 请求用户确认 */
  })
}
```

### 5. 重试机制

自动处理可重试的错误：

```typescript
const retry = SessionRetry.retryable(error)
if (retry !== undefined) {
  attempt++
  const delay = SessionRetry.delay(attempt, error)
  await SessionRetry.sleep(delay, input.abort)
  continue
}
```

## 错误处理流程

```typescript
try {
  const result = await processor.process(...)
} catch (e: any) {
  const error = MessageV2.fromError(e, { providerID: input.model.providerID })
  const retry = SessionRetry.retryable(error)

  if (retry !== undefined) {
    attempt++
    const delay = SessionRetry.delay(attempt, error)
    SessionStatus.set(input.sessionID, {
      type: "retry",
      attempt,
      message: retry,
      next: Date.now() + delay
    })
    await SessionRetry.sleep(delay, input.abort)
    continue
  }

  input.assistantMessage.error = error
  Bus.publish(Session.Event.Error, { sessionID, error })
}
```

## 性能优化

1. **上下文压缩**: 当 token 数量超过阈值时自动压缩历史消息
2. **文件缓存**: 文件读取和哈希缓存
3. **并行工具执行**: 支持多个工具并行执行
4. **增量更新**: 只更新变化的部分到 UI

## 总结

从 TUI 输入到代码完成的核心流程：

1. **输入**: TUI Prompt 组件捕获用户输入
2. **传输**: 通过 HTTP API 提交到服务器
3. **创建**: 创建用户消息并保存到存储
4. **循环**: SessionPrompt.loop() 处理主循环
5. **AI 调用**: LLM.stream() 调用 AI 模型
6. **流处理**: SessionProcessor 处理流式响应
7. **工具执行**: 根据工具调用执行相应操作
8. **权限检查**: Permission 系统控制操作权限
9. **快照跟踪**: Snapshot 跟踪文件变化
10. **UI 更新**: 通过 SSE 事件更新 TUI 界面

整个流程采用了事件驱动、流式处理、快照跟踪等设计模式，确保了系统的可扩展性、可维护性和用户体验。
