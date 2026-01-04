# CLI 目录结构划分和代码组织

## 一、整体设计理念

### 1.1 核心设计原则

OpenCode 项目采用**功能域驱动**（Domain-Driven Design）的目录结构划分，核心原则包括：

- **单一职责**：每个目录对应一个清晰的功能领域
- **高内聚低耦合**：相关代码集中在一起，模块间通过明确的接口通信
- **类型优先**：使用 TypeScript + Zod 强类型系统确保类型安全
- **命名空间组织**：使用 TypeScript 命名空间避免命名冲突，提供清晰的 API
- **扁平化工具**：工具目录保持扁平，每个文件专注单一功能

### 1.2 开源作者的思考过程

#### 第一阶段：识别核心功能域

作者首先识别出 AI CLI 工具的核心功能需求：

```
用户交互（CLI）
    ↓
命令解析 → 会话管理 → Agent 执行 → 工具调用 → 结果输出
    ↓           ↓           ↓          ↓
  配置管理    存储持久化   事件总线   文件系统
```

这种分层思考直接映射到目录结构。

#### 第二阶段：确定模块边界

每个模块的职责和边界被明确界定：

- **独立可测试**：每个模块可以独立测试
- **依赖清晰**：模块间的依赖关系单向流动
- **可扩展性**：通过插件和事件系统支持扩展

#### 第三阶段：设计接口和协议

通过 TypeScript 命名空间定义清晰的接口：

```typescript
export namespace Tool {
  export interface Info<Parameters extends z.ZodType, M extends Metadata> {
    id: string
    init: (ctx?: InitContext) => Promise<{
      description: string
      parameters: Parameters
      execute(...)
    }>
  }
}
```

这种设计确保了类型安全和 API 一致性。

---

## 二、目录结构详解

### 2.1 根目录文件

```
packages/opencode/
├── bin/              # CLI 入口脚本
├── src/              # 源代码
├── test/             # 测试文件
├── script/           # 构建和开发脚本
├── package.json      # 包配置
├── tsconfig.json     # TypeScript 配置
├── AGENTS.md         # Agent 开发指南
└── README.md         # 项目说明
```

**设计思考**：

- `bin/opencode`：CLI 可执行文件的入口，负责环境初始化
- `AGENTS.md`：为 Agent 开发者提供清晰的指导文档
- `script/`：将构建脚本与源代码分离，保持主目录简洁

### 2.2 核心功能域目录

```
src/
├── tool/             # 工具系统（文件操作、bash、网络等）
├── session/          # 会话管理和消息处理
├── agent/            # AI 代理配置和执行
├── server/           # HTTP API 服务器
├── cli/              # CLI 命令行界面
├── storage/          # 持久化存储层
├── bus/              # 事件总线系统
├── provider/         # AI 模型提供商
├── project/          # 项目管理（Git 集成等）
├── config/           # 配置管理
├── file/             # 文件系统操作
├── command/          # 命令解析
├── util/             # 通用工具函数
└── [其他功能域...]
```

---

## 三、核心模块深度解析

### 3.1 工具系统（tool/）

#### 目录结构

```
tool/
├── tool.ts           # Tool 定义和接口
├── registry.ts       # 工具注册表
├── bash.ts           # Bash 命令执行
├── edit.ts           # 文件编辑
├── read.ts           # 文件读取
├── write.ts          # 文件写入
├── glob.ts           # 文件查找
├── grep.ts           # 内容搜索
├── task.ts           # 子任务执行
├── webfetch.ts       # 网页获取
├── websearch.ts      # 网页搜索
├── [其他工具...]
└── *.txt             # 工具描述文本
```

#### 设计理念

**为什么这样划分？**

1. **统一接口**：所有工具都实现 `Tool.Info` 接口，确保一致性

```typescript
export function define<Parameters extends z.ZodType, Result extends Metadata>(
  id: string,
  init: Info<Parameters, Result>["init"],
): Info<Parameters, Result> {
  return {
    id,
    init: async (ctx) => {
      // 统一的参数验证和错误处理
    },
  }
}
```

2. **分离关注点**：
   - `tool.ts`：定义接口，不包含具体实现
   - `registry.ts`：管理工具的注册和查找
   - `*.ts`：每个文件一个工具，职责单一

3. **类型安全**：使用 Zod 定义工具参数，自动生成验证和文档

```typescript
// read.ts
export const Tool = Tool.define(
  "read",
  {
    description: "读取文件内容",
    parameters: z.object({
      filePath: z.string().describe("文件路径"),
      offset: z.number().optional(),
      limit: z.number().optional(),
    }),
    execute: async (args, ctx) => {
      const file = Bun.file(args.filePath)
      const content = await file.text()
      return { output: content, ... }
    }
  }
)
```

4. **配套文档**：每个工具有对应的 `.txt` 文件，用于生成工具描述

#### 如何借鉴？

在你的项目中创建工具系统时：

```typescript
// tools/registry.ts
export namespace Tool {
  export interface Info<Parameters extends z.ZodType> {
    id: string
    description: string
    parameters: Parameters
    execute(args: z.infer<Parameters>, ctx: Context): Promise<Result>
  }

  export function define<P>(config: ToolConfig<P>): Info<P> {
    return { ...config }
  }
}

// tools/my-tool.ts
export const MyTool = Tool.define({
  id: "my-tool",
  description: "我的工具",
  parameters: z.object({
    input: z.string(),
  }),
  execute: async (args, ctx) => {
    // 实现
  },
})
```

---

### 3.2 会话管理（session/）

#### 目录结构

```
session/
├── index.ts          # 主模块，导出公共 API
├── message.ts        # 消息定义
├── message-v2.ts     # V2 版本消息
├── prompt/           # 提示词模板
│   ├── prompt.ts
│   ├── compaction.txt
│   └── [其他提示词...]
├── status.ts         # 会话状态管理
├── summary.ts        # 会话摘要
├── compaction.ts     # 会话压缩
├── revert.ts         # 会话回滚
├── todo.ts           # TODO 列表管理
└── llm.ts            # LLM 调用
```

#### 设计理念

**为什么这样划分？**

1. **信息模型与操作分离**：
   - `message.ts`：数据结构定义
   - `index.ts`：CRUD 操作
   - `status.ts`：状态转换逻辑

2. **生命周期管理**：

   ```typescript
   export namespace Session {
     export const create = fn(...)
     export const get = fn(...)
     export const update = fn(...)
     export const remove = fn(...)
     export const fork = fn(...)
   }
   ```

3. **事件驱动**：每个操作都发布事件

```typescript
export const Event = {
  Created: BusEvent.define("session.created", z.object({ info: Info })),
  Updated: BusEvent.define("session.updated", z.object({ info: Info })),
  Deleted: BusEvent.define("session.deleted", z.object({ info: Info })),
}
```

4. **提示词集中管理**：所有提示词放在 `prompt/` 目录，易于维护

#### 如何借鉴？

设计会话系统时：

```typescript
// session/index.ts
export namespace Session {
  export const Info = z.object({
    id: z.string(),
    messages: z.array(Message),
    status: z.enum(["active", "completed", "failed"]),
    createdAt: z.number(),
  })

  export const create = fn(z.object({ ... }), async (input) => {
    const session = { ... }
    await Storage.write(["session", session.id], session)
    Bus.publish(Event.Created, { info: session })
    return session
  })
}

// session/status.ts
export namespace SessionStatus {
  export const transition = fn(..., (session, toStatus) => {
    // 状态转换逻辑
  })
}
```

---

### 3.3 Agent 系统（agent/）

#### 目录结构

```
agent/
├── agent.ts          # Agent 定义和配置
├── generate.txt      # Agent 生成提示词
├── prompt/
│   ├── compaction.txt
│   ├── explore.txt
│   ├── summary.txt
│   └── title.txt
```

#### 设计理念

**为什么这样划分？**

1. **配置驱动**：Agent 的所有配置都是声明式的

```typescript
export const Info = z.object({
  name: z.string(),
  description: z.string().optional(),
  mode: z.enum(["subagent", "primary", "all"]),
  tools: z.record(z.string(), z.boolean()),  // 哪些工具可用
  permission: z.object({ ... }),              // 权限配置
  model: z.object({ ... }).optional(),        // 模型配置
  prompt: z.string().optional(),              // 自定义提示词
})
```

2. **内置 Agent**：常见 Agent 内置在代码中

```typescript
const result: Record<string, Info> = {
  build: {
    name: "build",
    tools: { ...defaultTools },
    permission: agentPermission,
    mode: "primary",
  },
  explore: {
    name: "explore",
    description: "Fast agent specialized for exploring codebases",
    tools: { edit: false, write: false, ... },
    mode: "subagent",
  },
}
```

3. **动态生成**：支持通过 LLM 动态生成 Agent 配置

```typescript
export async function generate(input: { description: string }) {
  const result = await generateObject({
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

#### 如何借鉴？

设计 Agent 系统时：

```typescript
// agent/agent.ts
export namespace Agent {
  export const Info = z.object({
    name: z.string(),
    description: z.string(),
    capabilities: z.array(z.string()),
    tools: z.record(z.string(), z.boolean()),
  })

  export function register(config: AgentConfig) {
    // 注册 Agent
  }

  export async function execute(agentName: string, task: string) {
    const agent = await get(agentName)
    // 执行 Agent
  }
}
```

---

### 3.4 CLI 命令系统（cli/）

#### 目录结构

```
cli/
├── bootstrap.ts      # CLI 启动和初始化
├── ui.ts             # UI 渲染和样式
├── error.ts          # 错误格式化
├── network.ts        # 网络连接管理
├── upgrade.ts        # 版本升级
└── cmd/              # 具体命令实现
    ├── cmd.ts        # 命令类型定义
    ├── run.ts        # 主命令
    ├── generate.ts   # 生成命令
    ├── auth.ts       # 认证命令
    ├── agent.ts      # Agent 命令
    ├── tui/          # TUI 命令
    │   ├── attach.ts
    │   ├── spawn.ts
    │   └── thread.ts
    └── [其他命令...]
```

#### 设计理念

**为什么这样划分？**

1. **命令模块化**：每个命令一个文件

```typescript
// cli/cmd/run.ts
export const RunCommand = cmd({
  command: "run [message..]",
  describe: "run opencode with a message",
  builder: (yargs: Argv) => {
    return yargs
      .positional("message", { type: "string", array: true })
      .option("agent", { type: "string" })
      .option("model", { type: "string" })
  },
  handler: async (args) => {
    // 命令处理逻辑
  },
})
```

2. **统一入口**：在 `index.ts` 注册所有命令

```typescript
const cli = yargs(hideBin(process.argv))
  .command(RunCommand)
  .command(GenerateCommand)
  .command(AuthCommand)
  .command(AgentCommand)
  .command(UpgradeCommand)
  .command(UninstallCommand)
// ...
```

3. **关注点分离**：
   - `bootstrap.ts`：启动流程、初始化检查
   - `ui.ts`：UI 渲染、颜色、格式化
   - `error.ts`：错误处理、用户友好的错误信息

#### 如何借鉴？

设计 CLI 时：

```typescript
// cli/commands.ts
export const commands = [
  {
    name: "build",
    description: "Build the project",
    options: {
      watch: { type: "boolean", alias: "w" },
      config: { type: "string", alias: "c" },
    },
    handler: async (args) => {
      // 处理逻辑
    },
  },
]

// cli/index.ts
import { Command } from "commander"
import { commands } from "./commands"

const program = new Command()
commands.forEach((cmd) => {
  program
    .command(cmd.name)
    .description(cmd.description)
    .option(...cmd.options)
    .action(cmd.handler)
})
```

---

### 3.5 存储层（storage/）

#### 设计理念

```typescript
export namespace Storage {
  // 统一的存储接口
  export async function read<T>(key: string[]): Promise<T>
  export async function write<T>(key: string[], content: T): Promise<void>
  export async function update<T>(key: string[], fn: (draft: T) => void): Promise<T>
  export async function remove(key: string[]): Promise<void>
  export async function list(prefix: string[]): Promise<string[][]>
}
```

**特点**：

1. **键值对抽象**：使用路径数组作为键（如 `["session", projectId, sessionId]`）
2. **自动迁移**：内置数据迁移机制
3. **文件锁**：使用文件锁避免并发问题

#### 如何借鉴？

```typescript
// storage/storage.ts
export namespace Storage {
  const basePath = "./data"

  export async function read<T>(key: string[]): Promise<T> {
    const filePath = resolvePath(key)
    const content = await Bun.file(filePath).json()
    return content as T
  }

  export async function write<T>(key: string[], content: T): Promise<void> {
    const filePath = resolvePath(key)
    await ensureDir(path.dirname(filePath))
    await Bun.write(filePath, JSON.stringify(content, null, 2))
  }

  function resolvePath(key: string[]): string {
    return path.join(basePath, ...key) + ".json"
  }
}
```

---

### 3.6 事件总线（bus/）

#### 设计理念

```typescript
export namespace BusEvent {
  export function define<Type extends string, Properties extends ZodType>(
    type: Type,
    properties: Properties
  ) {
    const result = { type, properties }
    registry.set(type, result)
    return result
  }

  export function payloads() {
    return z.discriminatedUnion("type", [...])
  }
}

export namespace Bus {
  export function publish<T>(event: BusEvent.Definition, payload: T): void
  export function subscribe<T>(event: BusEvent.Definition, handler: (payload: T) => void): void
}
```

**特点**：

1. **类型安全**：通过 Zod 确保事件类型安全
2. **自动注册**：所有事件自动注册到全局事件注册表
3. **解耦通信**：模块间通过事件通信，不直接依赖

#### 如何借鉴？

```typescript
// bus/event.ts
export namespace Event {
  export const UserCreated = define(
    "user.created",
    z.object({
      userId: z.string(),
      email: z.string(),
    }),
  )

  export const UserUpdated = define(
    "user.updated",
    z.object({
      userId: z.string(),
      changes: z.record(z.any()),
    }),
  )
}

// bus/bus.ts
export const events = new EventEmitter()

export function publish<T>(event: Event.Definition<T>, payload: T) {
  events.emit(event.type, payload)
}

export function subscribe<T>(event: Event.Definition<T>, handler: (payload: T) => void) {
  events.on(event.type, handler)
}
```

---

### 3.7 工具库（util/）

#### 目录结构

```
util/
├── log.ts            # 日志系统
├── filesystem.ts     # 文件系统工具
├── lock.ts           # 文件锁
├── lazy.ts           # 懒加载
├── queue.ts          # 队列处理
├── eventloop.ts      # 事件循环
├── timeout.ts        # 超时处理
├── signal.ts         # 信号处理
├── token.ts          # Token 统计
├── fn.ts             # 函数工具
├── iife.ts           # 立即执行函数
├── defer.ts          # 延迟执行
├── scrap.ts          # 文本处理
├── wildcard.ts       # 通配符匹配
└── [其他工具...]
```

#### 设计理念

**为什么这样划分？**

1. **单文件单功能**：每个工具函数一个文件，职责明确
2. **可复用性**：所有工具都是纯函数或命名空间，易于复用
3. **类型安全**：每个工具都有明确的类型定义

例如，日志系统（log.ts）：

```typescript
export namespace Log {
  const Default = create({ service: "default" })

  export function create(tags?: Record<string, any>) {
    return {
      debug(message?: any, extra?: Record<string, any>) { ... },
      info(message?: any, extra?: Record<string, any>) { ... },
      error(message?: any, extra?: Record<string, any>) { ... },
      time(message: string, extra?: Record<string, any>) { ... },
    }
  }
}
```

#### 如何借鉴？

```typescript
// util/logger.ts
export namespace Logger {
  export const levels = ["DEBUG", "INFO", "WARN", "ERROR"] as const
  export type Level = (typeof levels)[number]

  export function create(service: string) {
    return {
      debug(message: string, data?: any) {
        log("DEBUG", service, message, data)
      },
      info(message: string, data?: any) {
        log("INFO", service, message, data)
      },
    }
  }

  function log(level: Level, service: string, message: string, data?: any) {
    const timestamp = new Date().toISOString()
    console.log(`${timestamp} [${level}] ${service}: ${message}`)
    if (data) console.log(JSON.stringify(data))
  }
}
```

---

## 四、代码组织模式

### 4.1 命名空间模式

OpenCode 大量使用 TypeScript 命名空间来组织代码：

```typescript
export namespace Tool {
  export interface Info<P extends z.ZodType, M extends Metadata> { ... }
  export function define<P>(...): Info<P> { ... }
  export namespace Context { ... }
}
```

**优点**：

1. **避免命名冲突**：不同模块可以有相同的方法名
2. **清晰的 API**：通过 `Tool.define()` 这种调用方式，语义清晰
3. **类型推导**：`Tool.Info` 中的类型参数可以推导到 `define` 的参数

**如何借鉴**：

```typescript
// 不要这样做（容易冲突）
export interface ToolInfo<P> { ... }
export function defineTool<P>(...): ToolInfo<P> { ... }

// 这样做（更清晰）
export namespace Tool {
  export interface Info<P> { ... }
  export function define<P>(...): Info<P> { ... }
}
```

---

### 4.2 工厂函数模式

使用工厂函数创建复杂对象：

```typescript
export namespace Log {
  export function create(tags?: Record<string, any>): Logger {
    return {
      debug(...) { ... },
      info(...) { ... },
    }
  }
}

// 使用
const logger = Log.create({ service: "my-service" })
logger.info("Hello")
```

**优点**：

1. **灵活配置**：可以传入配置参数
2. **封装细节**：使用者不需要了解内部实现
3. **便于测试**：可以创建 mock 实例

---

### 4.3 函数封装模式

使用 `fn()` 函数封装带有参数验证的函数：

```typescript
export namespace Storage {
  export const read = fn(Identifier.schema("session"), async (id) => {
    const result = await Storage.read<Info>(["session", Instance.project.id, id])
    return result as Info
  })
}

// fn 定义
export function fn<T extends z.ZodTypeAny, R>(
  schema: T,
  handler: (input: z.infer<T>) => Promise<R>,
): (input: z.infer<T>) => Promise<R> {
  return async (input) => {
    const validated = schema.parse(input)
    return handler(validated)
  }
}
```

**优点**：

1. **自动验证**：输入参数自动验证
2. **类型安全**：类型从 schema 自动推导
3. **声明式**：函数签名和验证逻辑在一起

---

### 4.4 事件驱动模式

模块间通过事件通信：

```typescript
// 发布事件
Bus.publish(Event.Created, { info: session })

// 订阅事件
Bus.subscribe(Event.Created, ({ info }) => {
  console.log("Session created:", info.id)
})
```

**优点**：

1. **解耦**：发布者和订阅者不直接依赖
2. **可扩展**：可以随时添加新的订阅者
3. **异步友好**：天然支持异步处理

---

## 五、如何借鉴到你的项目

### 5.1 评估你的项目需求

先问自己几个问题：

1. **功能复杂度**：是否需要多个独立的功能域？
2. **扩展性需求**：是否需要支持插件或扩展？
3. **类型安全**：是否需要严格的类型检查？
4. **模块通信**：模块间如何通信？（直接调用 vs 事件总线）

### 5.2 小型项目的简化方案

如果项目较小，可以简化结构：

```
src/
├── cli/              # CLI 命令
├── core/             # 核心逻辑
├── utils/            # 工具函数
└── types/            # 类型定义
```

**示例**：

```typescript
// core/session.ts
export type Session = {
  id: string
  messages: Message[]
}

export async function createSession(): Promise<Session> {
  return { id: generateId(), messages: [] }
}

// cli/run.ts
import { createSession } from "../core/session"

export async function run() {
  const session = await createSession()
  // ...
}
```

---

### 5.3 中型项目的标准方案

功能域划分：

```
src/
├── commands/         # CLI 命令
├── services/         # 业务服务
├── models/           # 数据模型
├── storage/          # 存储层
├── events/           # 事件系统
└── utils/            # 工具函数
```

**示例**：

```typescript
// models/session.ts
export namespace Session {
  export const Info = z.object({
    id: z.string(),
    status: z.enum(["active", "completed"]),
  })

  export function create(): Info {
    return { id: generateId(), status: "active" }
  }
}

// services/session-service.ts
export namespace SessionService {
  export async function create(): Promise<Session.Info> {
    const session = Session.create()
    await Storage.write(["session", session.id], session)
    Events.publish("session.created", session)
    return session
  }
}

// commands/run.ts
export const run = async () => {
  const session = await SessionService.create()
  // ...
}
```

---

### 5.4 大型项目的完整方案

参考 OpenCode 的完整结构：

```
src/
├── agent/            # AI 代理
├── command/          # 命令解析
├── config/           # 配置管理
├── storage/          # 持久化存储
├── bus/              # 事件总线
├── tool/             # 工具系统
├── session/          # 会话管理
├── cli/              # CLI 界面
├── server/           # API 服务器
├── provider/         # 外部服务提供商
├── util/             # 通用工具
└── [其他功能域...]
```

**关键点**：

1. **每个功能域一个命名空间**
2. **统一的接口和模式**
3. **事件驱动架构**
4. **类型优先设计**

---

### 5.5 实施步骤

#### 步骤 1：识别功能域

列出你的项目需要哪些功能域：

```
用户管理
订单处理
支付集成
通知系统
数据分析
...
```

#### 步骤 2：定义接口

为每个功能域定义清晰的接口：

```typescript
// user/index.ts
export namespace User {
  export const Info = z.object({
    id: z.string(),
    email: z.string(),
    name: z.string(),
  })

  export async function create(input: CreateUserInput): Promise<Info>
  export async function get(id: string): Promise<Info>
  export async function update(id: string, changes: UpdateUserInput): Promise<Info>
  export async function remove(id: string): Promise<void>
}
```

#### 步骤 3：选择架构模式

根据需求选择：

- **直接调用**：简单直接，适合小型项目
- **事件驱动**：解耦，适合中大型项目
- **依赖注入**：灵活，适合需要可测试性的项目

#### 步骤 4：实现工具库

先实现基础工具：

```typescript
// util/logger.ts
export namespace Logger {
  export function create(service: string) {
    return {
      debug(message: string, data?: any) { ... },
      info(message: string, data?: any) { ... },
      error(message: string, error?: Error) { ... },
    }
  }
}

// util/storage.ts
export namespace Storage {
  export async function read<T>(key: string[]): Promise<T> { ... }
  export async function write<T>(key: string[], data: T): Promise<void> { ... }
  export async function list(prefix: string[]): Promise<string[][]> { ... }
}
```

#### 步骤 5：迭代优化

- **关注点分离**：每个文件只做一件事
- **减少依赖**：模块间依赖尽可能少
- **统一风格**：保持代码风格一致

---

## 六、最佳实践总结

### 6.1 目录组织原则

✅ **推荐**：

```
src/
├── feature-1/        # 功能域
│   ├── index.ts      # 公共接口
│   ├── model.ts      # 数据模型
│   └── service.ts    # 业务逻辑
├── feature-2/
│   └── ...
└── shared/           # 共享代码
```

❌ **避免**：

```
src/
├── controllers/      # 按技术层划分
├── services/
├── models/
└── utils/
```

**理由**：按功能域划分更符合业务逻辑，减少跨层依赖。

---

### 6.2 文件命名原则

- **使用命名空间**：`session/index.ts` 而不是 `sessions.ts`
- **单文件单职责**：`bash.ts` 而不是 `tools.ts`
- **类型定义在主文件**：`tool.ts` 定义 `Tool.Info`
- **导出接口在 index.ts**：每个功能域有一个 `index.ts`

---

### 6.3 代码风格建议

```typescript
// ✅ 使用命名空间
export namespace Session {
  export const Info = z.object({ ... })
  export function create() { ... }
}

// ❌ 避免全局函数
export const SessionInfo = z.object({ ... })
export function createSession() { ... }

// ✅ 使用工厂函数
export const logger = Logger.create({ service: "my-service" })

// ❌ 避免直接构造
export const logger = new Logger({ service: "my-service" })
```

---

### 6.4 类型安全优先

```typescript
// ✅ 使用 Zod 定义类型
export const UserInput = z.object({
  name: z.string().min(1),
  email: z.string().email(),
})

export async function createUser(input: z.infer<typeof UserInput>) {
  const validated = UserInput.parse(input)
  // ...
}

// ❌ 避免任意类型
export async function createUser(input: any) {
  // 没有类型检查
}
```

---

## 七、总结

OpenCode 项目的目录结构设计体现了**功能域驱动**和**类型优先**的设计理念：

1. **按功能划分**：每个目录对应一个清晰的功能领域
2. **命名空间组织**：使用 TypeScript 命名空间提供清晰的 API
3. **统一接口**：每个功能域都有统一的接口和模式
4. **事件驱动**：模块间通过事件通信，保持解耦
5. **类型安全**：使用 Zod 确保类型安全

**借鉴要点**：

- ✅ **从小做起**：先识别核心功能域
- ✅ **保持简单**：避免过度设计
- ✅ **统一风格**：保持代码风格一致
- ✅ **类型优先**：充分利用 TypeScript 的类型系统
- ✅ **关注可维护性**：目录结构应该易于理解和导航

**记住**：没有"完美"的目录结构，只有"适合"你项目需求的目录结构。OpenCode 的结构可以作为参考，但要根据你的实际情况调整。
