# Zod 在 OpenCode 项目中的使用

## 1. 什么是 Zod？

Zod 是一个 TypeScript-first 的模式验证库，用于定义和验证数据结构。它可以在运行时和编译时都提供类型安全。

## 2. 为什么 OpenCode 需要 Zod？

OpenCode 是一个 AI 编码助手，需要处理大量来自用户、AI 模型和外部系统的数据。使用 Zod 主要有以下几个原因：

### 2.1 数据验证和类型安全

- **运行时验证**：确保传入的数据符合预期格式
- **类型推断**：自动推断 TypeScript 类型，避免手动维护
- **错误处理**：提供详细的错误信息，帮助调试

### 2.2 与 AI 工具调用集成

AI 模型需要调用各种工具（read, write, bash 等），这些工具的参数需要严格的验证，确保：

- AI 不会传入错误的参数类型
- 必需的参数不会缺失
- 参数值在合理的范围内

### 2.3 配置文件验证

OpenCode 支持复杂的配置系统（opencode.json），需要验证：

- 配置文件格式是否正确
- 必需字段是否存在
- 字段值是否有效（如颜色格式、路径等）

### 2.4 消息和事件系统的结构化

- 定义消息的结构（UserMessage, AssistantMessage）
- 定义消息部分的结构（TextPart, ToolPart, FilePart 等）
- 定义事件的结构
- 确保数据库存储和读取的数据一致性

## 3. Zod 的实际使用场景

### 3.1 工具参数定义和验证

**文件位置**：`packages/opencode/src/tool/read.ts`

```typescript
export const ReadTool = Tool.define("read", {
  description: DESCRIPTION,
  parameters: z.object({
    filePath: z.string().describe("The path to the file to read"),
    offset: z.coerce.number().describe("The line number to start reading from (0-based)").optional(),
    limit: z.coerce.number().describe("The number of lines to read (defaults to 2000)").optional(),
  }),
  async execute(params, ctx) {
    // 在这里，params 已经被 Zod 验证过，类型安全
    let filepath = params.filePath
    // ...
  },
})
```

**作用**：

- 定义 `read` 工具需要的参数结构
- `z.string()` 确保 filePath 是字符串
- `z.coerce.number()` 自动将字符串转换为数字
- `.optional()` 标记可选参数
- AI 调用工具时，Zod 会自动验证参数

### 3.2 工具注册和参数验证

**文件位置**：`packages/opencode/src/tool/tool.ts`

```typescript
export function define<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: Info<Parameters, Result>["init"] | Awaited<ReturnType<Info<Parameters, Result>["init"]>>,
): Info<Parameters, Result> {
  return {
    id,
    init: async (ctx) => {
      const toolInfo = init instanceof Function ? await init(ctx) : init
      const execute = toolInfo.execute
      toolInfo.execute = (args, ctx) => {
        try {
          // 在执行工具之前，先验证参数
          toolInfo.parameters.parse(args)
        } catch (error) {
          if (error instanceof z.ZodError && toolInfo.formatValidationError) {
            throw new Error(toolInfo.formatValidationError(error), { cause: error })
          }
          throw new Error(
            `The ${id} tool was called with invalid arguments: ${error}.\nPlease rewrite the input so it satisfies the expected schema.`,
            { cause: error },
          )
        }
        return execute(args, ctx)
      }
      return toolInfo
    },
  }
}
```

**作用**：

- 在工具执行前自动验证参数
- 提供友好的错误消息给 AI
- 如果验证失败，AI 会根据错误消息重新尝试

### 3.3 消息结构定义

**文件位置**：`packages/opencode/src/session/message.ts`

```typescript
export const ToolCall = z
  .object({
    state: z.literal("call"),
    step: z.number().optional(),
    toolCallId: z.string(),
    toolName: z.string(),
    args: z.custom<Required<unknown>>(),
  })
  .meta({
    ref: "ToolCall",
  })
export type ToolCall = z.infer<typeof ToolCall>

export const ToolResult = z
  .object({
    state: z.literal("result"),
    step: z.number().optional(),
    toolCallId: z.string(),
    toolName: z.string(),
    args: z.custom<Required<unknown>>(),
    result: z.string(),
  })
  .meta({
    ref: "ToolResult",
  })
export type ToolResult = z.infer<typeof ToolResult>

export const ToolInvocation = z.discriminatedUnion("state", [ToolCall, ToolPartialCall, ToolResult])
```

**作用**：

- 定义工具调用、工具结果等消息的结构
- 使用 `z.literal()` 确保字符串字面量类型安全
- 使用 `z.discriminatedUnion()` 创建可区分联合类型
- 使用 `z.infer()` 自动推断 TypeScript 类型

### 3.4 消息部分（Parts）定义

**文件位置**：`packages/opencode/src/session/message-v2.ts`

```typescript
export const TextPart = PartBase.extend({
  type: z.literal("text"),
  text: z.string(),
  synthetic: z.boolean().optional(),
  ignored: z.boolean().optional(),
  time: z
    .object({
      start: z.number(),
      end: z.number().optional(),
    })
    .optional(),
  metadata: z.record(z.string(), z.any()).optional(),
}).meta({
  ref: "TextPart",
})

export const FilePart = PartBase.extend({
  type: z.literal("file"),
  mime: z.string(),
  filename: z.string().optional(),
  url: z.string(),
  source: FilePartSource.optional(),
}).meta({
  ref: "FilePart",
})

export const Part = z
  .discriminatedUnion("type", [
    TextPart,
    SubtaskPart,
    ReasoningPart,
    FilePart,
    ToolPart,
    StepStartPart,
    StepFinishPart,
    SnapshotPart,
    PatchPart,
    AgentPart,
    RetryPart,
    CompactionPart,
  ])
  .meta({
    ref: "Part",
  })
```

**作用**：

- 定义消息可以包含的各种部分类型
- 使用 `z.discriminatedUnion()` 实现类型安全的联合
- 每个 Part 都有 `type` 字段用于区分
- 使用 `.meta()` 添加元数据引用（用于文档生成）

### 3.5 配置文件验证

**文件位置**：`packages/opencode/src/config/config.ts`

```typescript
export const Agent = z
  .object({
    model: z.string().optional(),
    temperature: z.number().optional(),
    top_p: z.number().optional(),
    prompt: z.string().optional(),
    tools: z.record(z.string(), z.boolean()).optional(),
    disable: z.boolean().optional(),
    description: z.string().optional().describe("Description of when to use the agent"),
    mode: z.enum(["subagent", "primary", "all"]).optional(),
    color: z
      .string()
      .regex(/^#[0-9a-fA-F]{6}$/, "Invalid hex color format")
      .optional()
      .describe("Hex color code for the agent (e.g., #FF5733)"),
    maxSteps: z
      .number()
      .int()
      .positive()
      .optional()
      .describe("Maximum number of agentic iterations before forcing text-only response"),
    permission: z
      .object({
        edit: Permission.optional(),
        bash: z.union([Permission, z.record(z.string(), Permission)]).optional(),
        skill: z.union([Permission, z.record(z.string(), Permission)]).optional(),
        webfetch: Permission.optional(),
        doom_loop: Permission.optional(),
        external_directory: Permission.optional(),
      })
      .optional(),
  })
  .catchall(z.any())
  .meta({
    ref: "AgentConfig",
  })
```

**作用**：

- 验证 Agent 配置的格式
- `z.enum()` 限制字段只能是特定值
- `z.regex()` 验证颜色格式（如 #FF5733）
- `z.record(z.string(), z.boolean())` 定义键值对类型
- `.catchall(z.any())` 允许额外字段（用于扩展）
- `.describe()` 添加字段说明（用于生成文档）

### 3.6 错误类型定义

**文件位置**：`packages/opencode/src/session/message.ts` 和 `packages/opencode/src/session/message-v2.ts`

```typescript
// message.ts
export const AuthError = NamedError.create(
  "ProviderAuthError",
  z.object({
    providerID: z.string(),
    message: z.string(),
  }),
)

// message-v2.ts
export const APIError = NamedError.create(
  "APIError",
  z.object({
    message: z.string(),
    statusCode: z.number().optional(),
    isRetryable: z.boolean(),
    responseHeaders: z.record(z.string(), z.string()).optional(),
    responseBody: z.string().optional(),
    metadata: z.record(z.string(), z.string()).optional(),
  }),
)
```

**作用**：

- 定义结构化的错误类型
- 包含详细的错误信息
- 可用于错误处理和恢复

### 3.7 事件系统

**文件位置**：`packages/opencode/src/bus/bus-event.ts`

```typescript
export function define<Type extends string, Properties extends ZodType>(type: Type, properties: Properties) {
  const result = {
    type,
    properties,
  }
  registry.set(type, result)
  return result
}

export const Event = {
  Updated: BusEvent.define(
    "message.updated",
    z.object({
      info: Info,
    }),
  ),
  PartUpdated: BusEvent.define(
    "message.part.updated",
    z.object({
      part: Part,
      delta: z.string().optional(),
    }),
  ),
}
```

**作用**：

- 定义事件的结构
- 注册事件类型
- 确保事件数据的一致性

### 3.8 与 AI SDK 集成（结构化输出）

**文件位置**：`packages/opencode/src/agent/agent.ts`

```typescript
export async function generate(input: { description: string; model?: { providerID: string; modelID: string } }) {
  const result = await generateObject({
    temperature: 0.3,
    messages: [
      {
        role: "user",
        content: `Create an agent configuration based on this request: "${input.description}".\n\nIMPORTANT: The following identifiers already exist and must NOT be used: ${existing.map((i) => i.name).join(", ")}\n  Return ONLY as JSON object, no other text, do not wrap in backticks`,
      },
    ],
    model: language,
    schema: z.object({
      identifier: z.string(),
      whenToUse: z.string(),
      systemPrompt: z.string(),
    }),
  })
  return result.object
}
```

**作用**：

- 使用 Zod schema 指导 AI 生成结构化输出
- 确保 AI 返回的 JSON 符合预期格式
- `generateObject` 函数会自动将 Zod schema 转换为 JSON Schema 发送给 AI

### 3.9 插件工具注册

**文件位置**：`packages/opencode/src/tool/registry.ts`

```typescript
function fromPlugin(id: string, def: ToolDefinition): Tool.Info {
  return {
    id,
    init: async () => ({
      parameters: z.object(def.args),
      description: def.description,
      execute: async (args, ctx) => {
        const result = await def.execute(args as any, ctx)
        return {
          title: "",
          output: result,
          metadata: {},
        }
      },
    }),
  }
}
```

**作用**：

- 将插件的工具定义转换为 OpenCode 的工具格式
- 使用 Zod 验证插件的工具参数

### 3.10 存储数据验证

**文件位置**：`packages/opencode/src/storage/storage.ts`

虽然 `Storage.read` 函数本身不直接使用 Zod 进行验证，但在从存储读取数据后，使用方通常会用 Zod schema 来验证数据：

```typescript
export async function read<T>(key: string[]) {
  const dir = await state().then((x) => x.dir)
  const target = path.join(dir, ...key) + ".json"
  return withErrorHandling(async () => {
    using _ = await Lock.read(target)
    const result = await Bun.file(target).json()
    return result as T
  })
}
```

**作用**：

- 提供基础的数据读取功能
- 使用方可以在读取后用 Zod schema 验证数据
- 防止数据损坏导致的运行时错误

## 4. Zod 的核心功能使用

### 4.1 类型推断

```typescript
const schema = z.object({
  name: z.string(),
  age: z.number(),
})

type Person = z.infer<typeof schema>
// 等同于:
// type Person = {
//   name: string
//   age: number
// }
```

### 4.2 字面量类型

```typescript
const ToolState = z.discriminatedUnion("status", [
  z.object({ status: z.literal("pending") }),
  z.object({ status: z.literal("running") }),
  z.object({ status: z.literal("completed") }),
])
```

### 4.3 可选字段和默认值

```typescript
z.object({
  required: z.string(),
  optional: z.string().optional(),
  withDefault: z.string().default("default value"),
})
```

### 4.4 类型转换

```typescript
z.coerce.number() // 自动将字符串转换为数字
z.coerce.boolean() // 自动将字符串 "true"/"false" 转换为布尔值
```

### 4.5 联合类型

```typescript
z.union([z.string(), z.number()])  // 字符串或数字
z.discriminatedUnion("type", [...]) // 可区分联合（根据某个字段区分）
```

### 4.6 正则表达式验证

```typescript
z.string().regex(/^#[0-9a-fA-F]{6}$/, "Invalid hex color")
```

### 4.7 记录类型（键值对）

```typescript
z.record(z.string(), z.boolean()) // Record<string, boolean>
```

### 4.8 自定义验证

```typescript
z.string().refine((val) => val.length > 0, "Cannot be empty")
```

### 4.9 错误处理

```typescript
const result = schema.safeParse(data)
if (!result.success) {
  console.log(result.error.errors)
  // [
  //   { path: ["name"], message: "Required" }
  // ]
}
```

### 4.10 元数据

```typescript
z.object({
  name: z.string(),
}).meta({
  ref: "Person", // 用于文档生成
  description: "A person",
})
```

## 5. Zod 在 OpenCode 中的价值总结

### 5.1 类型安全

- 在编译时和运行时都提供类型保护
- 避免手动维护 TypeScript 类型
- 减少类型错误

### 5.2 数据验证

- 确保所有外部输入都经过验证
- 提供清晰的错误信息
- 支持复杂的验证逻辑

### 5.3 AI 集成

- 定义工具参数，指导 AI 正确调用
- 生成结构化输出，确保 AI 返回符合预期的数据
- 提供错误反馈，帮助 AI 纠正错误

### 5.4 配置管理

- 验证配置文件格式
- 提供配置字段说明
- 支持配置文档自动生成

### 5.5 数据一致性

- 确保数据库中的数据格式一致
- 定义消息和事件的结构
- 防止数据损坏

### 5.6 开发体验

- 类型推断减少重复代码
- 自补全支持
- 清晰的 API 设计

## 6. 相关依赖

在 `package.json` 中：

```json
{
  "dependencies": {
    "zod": "catalog:",
    "zod-to-json-schema": "3.24.5"
  }
}
```

- **zod**: 核心 Zod 库
- **zod-to-json-schema**: 将 Zod schema 转换为 JSON Schema（用于与 AI SDK 集成）

## 7. 最佳实践

1. **使用 `describe()` 添加字段说明**：这有助于生成文档和 AI 理解字段含义
2. **使用 `meta()` 添加元数据**：用于文档生成和调试
3. **使用 `safeParse()` 进行验证**：避免异常，处理验证失败的情况
4. **使用 `infer()` 推断类型**：避免重复定义类型
5. **使用 `discriminatedUnion()` 替代 `union()`**：获得更好的类型推断
6. **使用 `refine()` 进行自定义验证**：处理复杂的验证逻辑
7. **使用 `catchall()` 允许扩展**：在配置 schema 中允许额外字段

## 8. 总结

Zod 在 OpenCode 项目中扮演着至关重要的角色，它不仅仅是一个数据验证库，更是整个项目类型系统和数据流的基石。通过与 AI SDK 的深度集成，Zod 确保了：

- AI 调用工具时参数正确
- AI 生成的输出符合预期格式
- 配置文件有效且可解析
- 消息和事件系统一致可靠

这种集成使得 OpenCode 能够安全、可靠地与 AI 模型交互，同时提供出色的开发体验和类型安全。
