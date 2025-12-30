# SDK 包的作用

## 什么是 SDK？

**SDK (Software Development Kit)** 软件开发工具包，是一套开发工具、库、文档和代码示例的集合，旨在帮助开发者更高效地与特定的软件框架、平台或服务进行交互。

SDK 通常包含：

- **API 库**：预封装的函数和类，简化与服务的通信
- **类型定义**：类型提示，提升开发体验（如 TypeScript）
- **文档和示例**：帮助开发者快速上手
- **工具函数**：简化常见操作

## OpenCode SDK 的作用

OpenCode SDK (`@opencode-ai/sdk`) 是一个**自动化生成的 TypeScript/TypeScript 客户端 SDK**，用于与 OpenCode 服务器进行 HTTP API 交互。

### 核心功能

#### 1. 客户端 (`client.ts`)

提供类型安全的 HTTP 客户端，通过 `createOpencodeClient` 函数创建：

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({
  baseUrl: "http://localhost:4096",
  directory: "/path/to/project",
})

// 调用 API
const session = await client.session.create()
const files = await client.file.list()
```

**特点**：

- 自动处理 HTTP 请求（GET、POST、PUT、DELETE 等）
- 支持 Server-Sent Events (SSE) 用于实时事件订阅
- 自动设置必要的请求头（如 `x-opencode-directory`）
- 完整的 TypeScript 类型支持

#### 2. 服务器 (`server.ts`)

启动和管理 OpenCode 服务器实例：

```typescript
import { createOpencodeServer } from "@opencode-ai/sdk"

const server = await createOpencodeServer({
  hostname: "127.0.0.1",
  port: 4096,
  timeout: 5000,
  config: {
    logLevel: "INFO",
    model: "anthropic/claude-3-5-sonnet",
  },
})

console.log("Server running at:", server.url)

// 清理
server.close()
```

**功能**：

- 使用 `child_process.spawn` 启动 `opencode serve` 命令
- 等待服务器启动并解析监听 URL
- 支持配置传递（通过环境变量）
- 提供 `close()` 方法优雅关闭服务器

#### 3. TUI (`server.ts`)

启动 OpenCode 的终端用户界面：

```typescript
import { createOpencodeTui } from "@opencode-ai/sdk"

const tui = createOpencodeTui({
  project: "my-project",
  model: "anthropic/claude-3-5-sonnet",
  session: "session-id",
  agent: "plan",
})

// 清理
tui.close()
```

#### 4. 统一接口 (`index.ts`)

提供 `createOpencode` 函数，同时创建服务器和客户端：

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode({
  hostname: "127.0.0.1",
  port: 4096,
})

// 使用 client 调用 API
await client.session.create()
```

## SDK 的 API 覆盖

SDK 基于 OpenAPI 规范自动生成，提供了以下模块：

### 核心模块

- **Session**: 会话管理（创建、删除、查询、发送消息、执行命令等）
- **File**: 文件操作（列出、读取、状态）
- **Pty**: 终端管理（创建、连接、更新、删除）
- **Config**: 配置管理（获取、更新）
- **Tool**: 工具管理（列出工具和工具 ID）
- **Instance**: 实例管理（清理实例）

### 扩展模块

- **Project**: 项目管理（列出、获取当前项目）
- **Provider**: AI 提供商管理（列表、认证、OAuth 作为子模块）
- **Find**: 查找功能（文本、文件、符号）
- **Command**: 命令列表

### 系统模块

- **Mcp**: MCP（Model Context Protocol）服务器管理
- **Lsp**: LSP（Language Server Protocol）状态
- **Formatter**: 格式化工具状态
- **App**: 应用日志和代理列表（通过 `app.agents()` 获取）
- **Path**: 路径信息
- **Vcs**: 版本控制信息
- **Auth**: 认证管理

### 交互模块

- **Tui**: TUI 控制（显示提示、执行命令、显示通知等）
- **Control**: TUI 控制队列（next、response）
- **Global**: 全局事件订阅
- **Event**: 会话事件订阅

## 代码生成机制

SDK 使用 `@hey-api/openapi-ts` 从 OpenAPI 规范（`openapi.json`）自动生成代码：

```
openapi.json → @hey-api/openapi-ts → gen/ 目录
```

**生成的文件结构**：

- `src/v2/gen/sdk.gen.ts`: 主 SDK 类定义
- `src/v2/gen/types.gen.ts`: TypeScript 类型定义
- `src/v2/gen/client/`: HTTP 客户端实现
- `src/v2/gen/core/`: 核心工具（序列化、认证等）

> 注意：代码目前只生成到 v2 目录。v1 的 `src/gen/` 目录是旧版本，可能不再维护。

**优势**：

- **自动化**: API 更新后自动重新生成 SDK
- **类型安全**: 完整的 TypeScript 类型覆盖
- **一致性**: SDK 与 OpenAPI 规范始终保持同步
- **易维护**: 减少手动维护代码的工作量

## 版本管理

SDK 提供两个版本：

### v1 版本

- 导入路径：`@opencode-ai/sdk`
- 文件位置：`src/` 目录
- 状态：可能已废弃，不再更新

### v2 版本

- 导入路径：`@opencode-ai/sdk/v2`
- 文件位置：`src/v2/` 目录
- 状态：当前活跃版本，持续更新
- 差异：gen 目录更完整，API 覆盖更全面

```typescript
// v1
import { createOpencodeClient } from "@opencode-ai/sdk"

// v2
import { createOpencodeClient } from "@opencode-ai/sdk/v2"
```

## 使用示例

### 示例 1: 批量处理文件

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

const files = ["file1.ts", "file2.ts", "file3.ts"]

for (const file of files) {
  const session = await client.session.create()
  await client.session.prompt({
    path: { id: session.data.id },
    body: {
      parts: [
        {
          type: "file",
          mime: "text/plain",
          url: `file://${file}`,
        },
        {
          type: "text",
          text: "Write tests for this file.",
        },
      ],
    },
  })
}
```

### 示例 2: 订阅事件

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

const eventStream = await client.event.subscribe()

for await (const event of eventStream) {
  console.log("Event:", event.type, event.properties)
}
```

### 示例 3: 启动服务器并使用

```typescript
import { createOpencode } from "@opencode-ai/sdk"

const { client, server } = await createOpencode({
  port: 4096,
  config: {
    model: "anthropic/claude-3-5-sonnet",
    logLevel: "DEBUG",
  },
})

// 使用 client
const session = await client.session.create()

// 完成后清理
server.close()
```

## SDK 的价值

### 对用户

1. **简化集成**: 无需手动构建 HTTP 请求
2. **类型安全**: TypeScript 提供完整的类型提示
3. **开发效率**: 快速调用 OpenCode 的所有 API
4. **实时通信**: 支持 SSE 实时事件订阅
5. **完整覆盖**: 提供所有 OpenCode API 的访问

### 对项目

1. **自动化**: 基于代码生成，减少手动维护
2. **可扩展**: 轻松添加新功能到 SDK
3. **文档化**: 类型定义即文档
4. **跨平台**: 支持任何能运行 JavaScript/TypeScript 的环境
5. **多端支持**: Web、Node.js、桌面应用、服务器等

## 技术细节

### 自定义 Fetch

SDK 使用自定义的 `fetch` 实现来禁用默认超时：

```typescript
const customFetch = (req) => {
  req.timeout = false
  return fetch(req)
}
```

这允许长时间运行的操作（如 AI 响应）不会被超时中断。

### 目录头

当传递 `directory` 参数时，SDK 会自动添加请求头：

```typescript
headers: {
  "x-opencode-directory": "/path/to/project"
}
```

这允许客户端指定要操作的项目目录。

### 配置传递

服务器和 TUI 的配置通过环境变量 `OPENCODE_CONFIG_CONTENT` 传递：

```typescript
env: {
  ...process.env,
  OPENCODE_CONFIG_CONTENT: JSON.stringify(options.config)
}
```

## 总结

OpenCode SDK 是一个强大的工具，它：

1. **抽象复杂性**: 隐藏了 HTTP 通信的细节
2. **提供类型安全**: 完整的 TypeScript 类型支持
3. **自动化生成**: 基于代码生成，易于维护
4. **完整覆盖**: 提供所有 OpenCode API 的访问
5. **灵活使用**: 支持服务器启动、客户端调用和 TUI 启动

SDK 使得开发者可以轻松地在自己的应用中集成 OpenCode 的功能，无论是 Web 应用、桌面应用还是自定义的自动化脚本。
