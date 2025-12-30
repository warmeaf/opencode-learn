# SDK 自动生成机制

## 目录

- [概述](#概述)
- [为什么需要自动生成](#为什么需要自动生成)
- [核心技术](#核心技术)
- [生成流程详解](#生成流程详解)
- [OpenAPI 规范](#openapi-规范)
- [生成的文件结构](#生成的文件结构)
- [实际开发流程](#实际开发流程)
- [常见问题](#常见问题)
- [最佳实践](#最佳实践)

---

## 概述

OpenCode SDK 的**自动生成机制**是指通过工具根据后端 API 的 OpenAPI 规范，自动创建完整的 TypeScript 客户端 SDK，无需手动编写和维护 SDK 代码。

### 核心思想

```
后端 API 定义 (OpenAPI 规范)
        ↓
   自动生成工具
        ↓
   客户端 SDK 代码
```

**关键点**：

- ✅ 后端变更 → 重新生成 SDK
- ✅ 类型安全自动保证
- ✅ 无需手动维护
- ✅ 始终与 API 同步

---

## 为什么需要自动生成

### 手动编写 SDK 的痛点

#### 1. 维护成本高昂

假设手动编写 SDK 的 `Session` 模块：

```typescript
// ❌ 手动编写 - 需要逐个方法实现
class SessionModule {
  // 创建会话
  async create(options: { model: string }): Promise<{ id: string }> {
    const response = await fetch("/session", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(options),
    })
    return response.json()
  }

  // 删除会话
  async delete(id: string): Promise<void> {
    await fetch(`/session/${id}`, { method: "DELETE" })
  }

  // 查询会话状态
  async status(id: string): Promise<SessionStatus> {
    const response = await fetch(`/session/${id}/status`)
    return response.json()
  }

  // 发送消息
  async prompt(id: string, message: string): Promise<SessionResponse> {
    const response = await fetch(`/session/${id}/message`, {
      method: "POST",
      body: JSON.stringify({ message }),
    })
    return response.json()
  }

  // ... 需要编写 20+ 个方法
}
```

**问题**：

- 每个方法都要手动实现
- 后端新增一个 API，就要手动写一个方法
- 后端修改 API 参数，就要手动更新
- 容易遗漏某些 API

#### 2. 类型定义繁琐

```typescript
// ❌ 手动编写类型 - 容易出错
interface Session {
  id: string
  createdAt: Date
  updatedAt?: Date
  model: string
  status: "idle" | "running" | "completed" | "error"
  // ... 十几个字段
}

interface SessionCreateOptions {
  model: string
  config?: {
    temperature?: number
    maxTokens?: number
  }
}

interface SessionPromptOptions {
  parts: Array<{
    type: "text" | "file"
    text?: string
    url?: string
  }>
}

// ... 需要定义 50+ 个类型
```

**问题**：

- 后端修改字段，类型定义容易不同步
- 手动维护容易出错
- 难以保证 100% 准确

#### 3. 工作流程低效

```
手动编写 SDK 的流程：

后端 API 开发完成
    ↓
阅读 API 文档
    ↓
手动编写 SDK 方法 (数小时)
    ↓
手动编写类型定义 (数小时)
    ↓
编写单元测试 (数小时)
    ↓
发布 SDK

总计：需要 1-2 天
```

### 自动生成的优势

#### 1. 开发效率提升

```typescript
// ✅ 自动生成 - 只需定义 OpenAPI 规范
// openapi.json
{
  "paths": {
    "/session": {
      "post": {
        "operationId": "session.create",
        "summary": "Create a new session",
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "model": { "type": "string" }
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "id": { "type": "string" }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

运行生成命令：

```bash
bun run build
```

✅ 完成！所有方法和类型自动生成。

```
自动生成 SDK 的流程：

后端 API 开发完成
    ↓
更新 OpenAPI 规范 (几分钟)
    ↓
运行生成命令 (几秒钟)
    ↓
自动测试 (可选)

总计：10 分钟
```

**对比**：

- 手动编写：1-2 天
- 自动生成：10 分钟
- **效率提升：20-100 倍**

#### 2. 准确性保证

| 对比项     | 手动编写   | 自动生成   |
| ---------- | ---------- | ---------- |
| 方法完整性 | 容易遗漏   | 100% 完整  |
| 类型准确性 | 容易出错   | 100% 准确  |
| 同步性     | 容易过时   | 实时同步   |
| 测试覆盖   | 需要手动写 | 可自动生成 |

#### 3. 维护成本降低

```typescript
// 后端新增 API /session/fork
// 手动编写：
// 1. 阅读文档
// 2. 编写方法
// 3. 定义类型
// 4. 测试
// 总计：2-3 小时

// 自动生成：
// 1. 在 openapi.json 添加定义（5 分钟）
// 2. 运行 bun run build（1 分钟）
// 总计：6 分钟
```

---

## 核心技术

### OpenAPI 规范

OpenAPI 规范（以前叫 Swagger）是一种用于描述 RESTful API 的标准格式，可以：

- 定义 API 端点、请求/响应格式
- 描述参数、数据类型
- 提供文档和示例

**示例**：

```json
{
  "openapi": "3.1.1",
  "info": {
    "title": "OpenCode API",
    "version": "1.0.0"
  },
  "paths": {
    "/session/{id}": {
      "get": {
        "operationId": "session.get",
        "summary": "Get session details",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "schema": { "type": "string" }
          }
        ],
        "responses": {
          "200": {
            "description": "Session details",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "id": { "type": "string" },
                    "model": { "type": "string" },
                    "status": {
                      "type": "string",
                      "enum": ["idle", "running", "completed"]
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### @hey-api/openapi-ts 工具

[`@hey-api/openapi-ts`](https://heyapi.vercel.app/) 是一个强大的 OpenAPI 代码生成器，用于 OpenCode SDK 生成。

**主要功能**：

- 根据 OpenAPI 规范生成 TypeScript 代码
- 生成类型定义、客户端代码、SDK 类
- 支持自定义配置和插件

---

## 生成流程详解

### 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    后端 API 开发                             │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              1. 生成 OpenAPI 规范                            │
│     bun dev generate > openapi.json                        │
│                                                             │
│  从后端代码自动提取 API 定义，生成标准化的 JSON 文件          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              2. 配置生成器                                    │
│  packages/sdk/js/script/build.ts                            │
│                                                             │
│  - 输入：openapi.json                                       │
│  - 输出：src/v2/gen/                                        │
│  - 插件：typescript, sdk, client-fetch                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              3. 运行生成工具                                │
│  @hey-api/openapi-ts 根据配置生成代码                       │
│                                                             │
│  自动创建：                                                  │
│  - sdk.gen.ts (SDK 类)                                      │
│  - types.gen.ts (类型定义)                                  │
│  - client/ (HTTP 客户端)                                    │
│  - core/ (核心工具)                                         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              4. 格式化和编译                                 │
│  bun prettier --write src/v2/gen                           │
│  bun tsc                                                    │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│              ✅ 生成完成                                     │
│  SDK 已更新，可以发布                                        │
└─────────────────────────────────────────────────────────────┘
```

### 代码实现

查看构建脚本：

```typescript
// packages/sdk/js/script/build.ts
#!/usr/bin/env bun

import { $ } from "bun"
import path from "path"
import { createClient } from "@hey-api/openapi-ts"

// 步骤 1: 从后端生成 OpenAPI 规范
await $`bun dev generate > ${dir}/openapi.json`.cwd(
  path.resolve(dir, "../../opencode")
)

// 步骤 2: 配置生成器
await createClient({
  input: "./openapi.json",           // OpenAPI 规范文件
  output: {
    path: "./src/v2/gen",            // 输出目录
    tsConfigPath: path.join(dir, "tsconfig.json"),
    clean: true,                     // 清空旧文件
  },
  plugins: [
    {
      name: "@hey-api/typescript",   // TypeScript 支持
      exportFromIndex: false,
    },
    {
      name: "@hey-api/sdk",          // SDK 生成
      instance: "OpencodeClient",    // 类名
      exportFromIndex: false,
      auth: false,
      paramsStructure: "flat",       // 参数结构
    },
    {
      name: "@hey-api/client-fetch", // Fetch 客户端
      exportFromIndex: false,
      baseUrl: "http://localhost:4096",
    },
  ],
})

// 步骤 3: 格式化代码
await $`bun prettier --write src/gen`
await $`bun prettier --write src/v2`

// 步骤 4: 类型检查
await $`rm -rf dist`
await $`bun tsc`

// 步骤 5: 清理临时文件
await $`rm openapi.json`
```

---

## OpenAPI 规范

### 规范结构

OpenAPI 规范是一个 JSON 文件，包含 API 的完整定义：

```json
{
  "openapi": "3.1.1",              // OpenAPI 版本
  "info": {
    "title": "OpenCode API",
    "version": "1.0.0"
  },
  "servers": [                    // 服务器地址
    {
      "url": "http://localhost:4096",
      "description": "Development server"
    }
  ],
  "paths": {                      // API 端点
    "/session": {
      "get": {
        "operationId": "session.list",
        "summary": "List all sessions",
        "tags": ["Session"],
        "responses": { ... }
      },
      "post": {
        "operationId": "session.create",
        "summary": "Create a new session",
        "requestBody": { ... },
        "responses": { ... }
      }
    },
    "/session/{id}": {
      "get": {
        "operationId": "session.get",
        "parameters": [ ... ],
        "responses": { ... }
      }
    }
  },
  "components": {                 // 可复用组件
    "schemas": {                  // 数据模型
      "Session": {
        "type": "object",
        "properties": {
          "id": { "type": "string" },
          "model": { "type": "string" },
          "status": { "type": "string" }
        }
      }
    }
  }
}
```

### 实际例子：Session API

#### 定义 GET /session

```json
{
  "paths": {
    "/session": {
      "get": {
        "operationId": "session.list",
        "summary": "List all sessions",
        "description": "Returns a list of all active sessions",
        "tags": ["Session"],
        "responses": {
          "200": {
            "description": "List of sessions",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "$ref": "#/components/schemas/Session"
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### 定义 POST /session

```json
{
  "paths": {
    "/session": {
      "post": {
        "operationId": "session.create",
        "summary": "Create a new session",
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "$ref": "#/components/schemas/SessionCreateRequest"
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "Created session",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Session"
                }
              }
            }
          },
          "400": {
            "description": "Invalid request",
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Error"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### 定义数据模型

```json
{
  "components": {
    "schemas": {
      "Session": {
        "type": "object",
        "required": ["id", "model"],
        "properties": {
          "id": {
            "type": "string",
            "description": "Session ID"
          },
          "model": {
            "type": "string",
            "description": "AI model name"
          },
          "status": {
            "type": "string",
            "enum": ["idle", "running", "completed", "error"]
          },
          "createdAt": {
            "type": "string",
            "format": "date-time"
          }
        }
      },
      "SessionCreateRequest": {
        "type": "object",
        "properties": {
          "model": {
            "type": "string",
            "description": "AI model name"
          },
          "config": {
            "type": "object",
            "properties": {
              "temperature": {
                "type": "number",
                "minimum": 0,
                "maximum": 1
              },
              "maxTokens": {
                "type": "integer"
              }
            }
          }
        }
      }
    }
  }
}
```

---

## 生成的文件结构

### 完整目录结构

```
packages/sdk/js/src/v2/gen/
├── sdk.gen.ts           # SDK 主类（所有 API 方法）
├── types.gen.ts         # 所有类型定义
├── client.gen.ts        # 客户端配置
├── client/              # HTTP 客户端实现
│   ├── index.ts
│   ├── client.gen.ts    # 基础客户端
│   ├── types.gen.ts     # 客户端类型
│   └── utils.gen.ts     # 客户端工具
└── core/                # 核心工具
    ├── auth.gen.ts      # 认证处理
    ├── bodySerializer.gen.ts   # 请求体序列化
    ├── pathSerializer.gen.ts   # 路径参数处理
    ├── queryKeySerializer.gen.ts # 查询参数处理
    ├── serverSentEvents.gen.ts  # SSE 支持
    └── types.gen.ts     # 核心类型
```

### 关键文件详解

#### 1. sdk.gen.ts - SDK 主类

生成的 SDK 主类包含所有 API 模块：

```typescript
// This file is auto-generated by @hey-api/openapi-ts

import { client as _heyApiClient } from "./client.gen.js"
import type { Options, Client, TDataShape } from "./client/index.js"

// 基础客户端类
class _HeyApiClient {
  protected _client: Client = _heyApiClient
  constructor(args?: { client?: Client }) {
    if (args?.client) this._client = args.client
  }
}

// Session 模块
class Session extends _HeyApiClient {
  /**
   * List all sessions
   */
  public list<ThrowOnError extends boolean = false>(options?) {
    return (options?.client ?? this._client).get<{ data: Session[] }, unknown, ThrowOnError>({
      url: "/session",
      ...options,
    })
  }

  /**
   * Create a new session
   */
  public create<ThrowOnError extends boolean = false>(options) {
    return (options?.client ?? this._client).post<{ data: Session }, Error, ThrowOnError>({
      url: "/session",
      body: options.body,
      ...options,
    })
  }

  /**
   * Get session details
   */
  public get<ThrowOnError extends boolean = false>(options) {
    return (options?.client ?? this._client).get<{ data: Session }, Error, ThrowOnError>({
      url: "/session/{id}",
      path: options.path,
      ...options,
    })
  }

  // ... 更多方法
}

// File 模块
class File extends _HeyApiClient {
  public list(options?) {
    /* ... */
  }
  public read(options) {
    /* ... */
  }
  public status(options) {
    /* ... */
  }
}

// ... 其他模块

// 主 SDK 类
export class OpencodeClient extends _HeyApiClient {
  global = new Global({ client: this._client })
  session = new Session({ client: this._client })
  file = new File({ client: this._client })
  project = new Project({ client: this._client })
  pty = new Pty({ client: this._client })
  config = new Config({ client: this._client })
  tool = new Tool({ client: this._client })
  instance = new Instance({ client: this._client })
  path = new Path({ client: this._client })
  vcs = new Vcs({ client: this._client })
  command = new Command({ client: this._client })
  provider = new Provider({ client: this._client })
  find = new Find({ client: this._client })
  app = new App({ client: this._client })
  mcp = new Mcp({ client: this._client })
  lsp = new Lsp({ client: this._client })
  formatter = new Formatter({ client: this._client })
  tui = new Tui({ client: this._client })
  auth = new Auth({ client: this._client })
  event = new Event({ client: this._client })
}
```

**特点**：

- 每个模块一个类
- 所有方法都有完整类型
- 支持泛型和错误处理
- 自动文档注释

#### 2. types.gen.ts - 类型定义

包含所有 API 的类型定义：

```typescript
// This file is auto-generated by @hey-api/openapi-ts

export type SessionListData = void
export type SessionListResponses = { data: Session[] }
export type SessionCreateData = void
export type SessionCreateResponses = { data: Session }
export type SessionCreateErrors = { detail: string }
export type SessionGetData = void
export type SessionGetResponses = { data: Session }
export type SessionGetErrors = { detail: string }

// Session 数据模型
export type Session = {
  id: string
  model: string
  status: "idle" | "running" | "completed" | "error"
  createdAt: string
  updatedAt?: string
  config?: SessionConfig
}

// Session 创建请求
export type SessionCreateRequest = {
  model: string
  config?: {
    temperature?: number
    maxTokens?: number
  }
}

// Session 配置
export type SessionConfig = {
  temperature?: number
  maxTokens?: number
  systemPrompt?: string
}

// ... 更多类型
```

**特点**：

- 从 OpenAPI 规范自动生成
- 100% 类型安全
- 包含所有请求和响应类型
- 支持联合类型和枚举

#### 3. client/client.gen.ts - HTTP 客户端

自动生成的 HTTP 客户端：

```typescript
// This file is auto-generated by @hey-api/openapi-ts

import type { ClientOptions, Client, HttpRequest, HttpResponse } from "./index.js"

export const client = async (request: HttpRequest): Promise<HttpResponse> => {
  // 创建请求
  const { url, ...options } = request

  // 设置默认值
  const headers = {
    "Content-Type": "application/json",
    ...options.headers,
  }

  // 发送请求
  const response = await fetch(url, {
    ...options,
    headers,
  })

  // 返回响应
  return {
    data: await response.json(),
    response,
  }
}
```

**特点**：

- 标准的 HTTP 客户端
- 支持自定义配置
- 统一的错误处理
- 类型安全的响应

#### 4. core/ - 核心工具

各种序列化和处理工具：

```typescript
// pathSerializer.gen.ts - 路径参数序列化
export function pathSerializer<T extends Record<string, unknown>>(path: T, url: string): string {
  // 替换路径参数 /session/{id} -> /session/123
  return Object.entries(path).reduce((acc, [key, value]) => {
    return acc.replace(`{${key}}`, encodeURIComponent(String(value)))
  }, url)
}

// bodySerializer.gen.ts - 请求体序列化
export function bodySerializer<T>(body: T): T {
  return body
}

// queryKeySerializer.gen.ts - 查询参数序列化
export function queryKeySerializer<T>(query: T): string {
  // 处理查询参数 ?limit=10&offset=0
  return Object.entries(query)
    .filter(([, value]) => value !== undefined)
    .map(([key, value]) => `${key}=${encodeURIComponent(String(value))}`)
    .join("&")
}
```

---

## 实际开发流程

### 场景 1：后端新增 API

假设后端新增一个 API：`POST /session/{id}/fork`（fork 会话）

#### 步骤 1：更新 OpenAPI 规范

在 `packages/opencode` 中更新 API 定义：

```json
// openapi.json
{
  "paths": {
    "/session/{id}/fork": {
      "post": {
        "operationId": "session.fork",
        "summary": "Fork an existing session",
        "parameters": [
          {
            "name": "id",
            "in": "path",
            "required": true,
            "schema": { "type": "string" }
          }
        ],
        "requestBody": {
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "model": { "type": "string" }
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "content": {
              "application/json": {
                "schema": {
                  "$ref": "#/components/schemas/Session"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

#### 步骤 2：运行生成命令

```bash
cd packages/sdk/js
bun run build
```

构建脚本会：

1. 从后端生成最新的 openapi.json
2. 使用 @hey-api/openapi-ts 生成代码
3. 格式化和类型检查

#### 步骤 3：自动生成的代码

SDK 自动获得新方法：

```typescript
// src/v2/gen/sdk.gen.ts（自动生成）
class Session extends _HeyApiClient {
  // ... 其他方法

  /**
   * Fork an existing session
   */
  public fork<ThrowOnError extends boolean = false>(options) {
    return (options?.client ?? this._client).post<{ data: Session }, Error, ThrowOnError>({
      url: "/session/{id}/fork",
      path: options.path,
      body: options.body,
      ...options,
    })
  }
}
```

类型定义自动添加：

```typescript
// src/v2/gen/types.gen.ts（自动生成）
export type SessionForkData = void
export type SessionForkResponses = { data: Session }
export type SessionForkErrors = { detail: string }
```

#### 步骤 4：使用新 API

开发者可以立即使用：

```typescript
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient({ baseUrl: "http://localhost:4096" })

// fork 会话
const forkedSession = await client.session.fork({
  path: { id: "session-123" },
  body: { model: "anthropic/claude-3-5-sonnet" },
})

console.log(forkedSession.data.id)
```

**总耗时：5-10 分钟**

### 场景 2：后端修改 API

假设后端修改 `/session` 的响应，新增 `forkedFrom` 字段。

#### 步骤 1：更新 OpenAPI 规范

```json
{
  "components": {
    "schemas": {
      "Session": {
        "type": "object",
        "properties": {
          "id": { "type": "string" },
          "model": { "type": "string" },
          "forkedFrom": {
            // 新增字段
            "type": "string",
            "description": "Original session ID if forked"
          }
        }
      }
    }
  }
}
```

#### 步骤 2：运行生成

```bash
cd packages/sdk/js
bun run build
```

#### 步骤 3：类型自动更新

```typescript
// types.gen.ts 自动更新
export type Session = {
  id: string
  model: string
  forkedFrom?: string // 自动添加
}
```

无需修改任何手写代码，类型自动更新！

### 场景 3：修复生成问题

假设生成的代码有问题，需要调整。

#### 步骤 1：修改生成配置

```typescript
// packages/sdk/js/script/build.ts
await createClient({
  input: "./openapi.json",
  output: {
    path: "./src/v2/gen",
    tsConfigPath: path.join(dir, "tsconfig.json"),
    clean: true,
  },
  plugins: [
    {
      name: "@hey-api/typescript",
      exportFromIndex: false,
    },
    {
      name: "@hey-api/sdk",
      instance: "OpencodeClient",
      exportFromIndex: false,
      auth: false,
      paramsStructure: "flat", // 修改为 flat 结构
    },
    {
      name: "@hey-api/client-fetch",
      exportFromIndex: false,
      baseUrl: "http://localhost:4096",
    },
  ],
})
```

#### 步骤 2：重新生成

```bash
bun run build
```

#### 步骤 3：验证

```bash
# 类型检查
bun tsc

# 运行测试
bun test
```

---

## 常见问题

### Q1: 为什么生成的代码和我手动写的不一样？

**问题**：

```typescript
// 我期望的代码
async createSession(model: string): Promise<Session> {
  const response = await fetch("/session", {
    method: "POST",
    body: JSON.stringify({ model })
  })
  return response.json()
}

// 生成的代码
public create<ThrowOnError extends boolean = false>(options?) {
  return (options?.client ?? this._client).post<SessionCreateResponses, SessionCreateErrors, ThrowOnError>({
    url: "/session",
    body: options?.body,
    ...options
  })
}
```

**原因**：

- 生成的代码更灵活，支持自定义配置
- 使用了泛型，支持类型安全的错误处理
- 支持自定义客户端实例

**解决方案**：
封装一个更简单的接口：

```typescript
// src/helper.ts
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient()

export async function createSession(model: string): Promise<Session> {
  const response = await client.session.create({
    body: { model },
  })
  return response.data
}
```

### Q2: 如何调试生成的代码？

**问题**：生成的代码太复杂，不知道哪里出错。

**解决方案**：

1. **查看 OpenAPI 规范**

   ```bash
   # 在 SDK 目录保留 openapi.json
   bun dev generate > openapi.json
   # 手动检查规范是否正确
   ```

2. **检查生成的代码**

   ```bash
   # 查看生成的文件
   cat src/v2/gen/sdk.gen.ts | grep "class Session" -A 20
   ```

3. **使用 TypeScript 检查**

   ```bash
   # 类型检查
   bun tsc --noEmit
   ```

4. **编写测试**

   ```typescript
   // test/sdk.test.ts
   import { createOpencodeClient } from "@opencode-ai/sdk"

   const client = createOpencodeClient()

   test("session.create", async () => {
     const response = await client.session.create({
       body: { model: "test-model" },
     })
     expect(response.data.id).toBeDefined()
   })
   ```

### Q3: 生成失败怎么办？

**常见原因**：

1. **OpenAPI 规范格式错误**

   ```json
   // ❌ 错误
   {
     "paths": {
       "/session": {
         "post": {
           "operationId": "session.create"
           // 缺少 requestBody 和 responses
         }
       }
     }
   }

   // ✅ 正确
   {
     "paths": {
       "/session": {
         "post": {
           "operationId": "session.create",
           "requestBody": { ... },
           "responses": { ... }
         }
       }
     }
   }
   ```

2. **类型冲突**

   ```json
   // ❌ 同名类型导致冲突
   {
     "components": {
       "schemas": {
         "Session": { ... },
         "session": { ... }  // 类型名冲突
       }
     }
   }
   ```

3. **循环引用**
   ```json
   // ❌ 类型互相引用
   {
     "components": {
       "schemas": {
         "A": {
           "properties": {
             "b": { "$ref": "#/components/schemas/B" }
           }
         },
         "B": {
           "properties": {
             "a": { "$ref": "#/components/schemas/A" }
           }
         }
       }
     }
   }
   ```

**解决方案**：

```bash
# 1. 验证 OpenAPI 规范
npm install -g @apidevtools/swagger-cli
swagger-cli validate openapi.json

# 2. 清理并重新生成
rm -rf src/v2/gen
bun run build

# 3. 查看详细错误日志
DEBUG=@hey-api/* bun run build
```

### Q4: 如何自定义生成的代码？

**场景**：想给所有 API 方法添加日志。

**方法 1：创建封装层**

```typescript
// src/wrapped-client.ts
import { createOpencodeClient } from "@opencode-ai/sdk"

const rawClient = createOpencodeClient()

function withLogging<T extends (...args: any[]) => Promise<any>>(fn: T): T {
  return (async (...args: any[]) => {
    console.log(`[SDK] Calling ${fn.name}`, args)
    const result = await fn(...args)
    console.log(`[SDK] ${fn.name} completed`)
    return result
  }) as T
}

// 封装所有方法
export const client = {
  session: {
    list: withLogging(rawClient.session.list.bind(rawClient.session)),
    create: withLogging(rawClient.session.create.bind(rawClient.session)),
    // ... 其他方法
  },
  // ... 其他模块
}
```

**方法 2：自定义插件**

```typescript
// script/build.ts
import { createClient } from "@hey-api/openapi-ts"

await createClient({
  input: "./openapi.json",
  output: { path: "./src/v2/gen" },
  plugins: [
    {
      name: "@hey-api/sdk",
      // ... 配置
    },
    {
      name: "custom-logging-plugin",
      setup({ onInit }) {
        onInit(() => {
          console.log("SDK generation started")
        })
      },
    },
  ],
})
```

### Q5: v1 和 v2 的区别？

**对比**：

| 特性         | v1         | v2            |
| ------------ | ---------- | ------------- |
| 生成目录     | `src/gen/` | `src/v2/gen/` |
| SDK 文件大小 | 36KB       | 70KB          |
| 更新状态     | 可能废弃   | 活跃更新      |
| 功能覆盖     | 基础功能   | 完整功能      |

**建议**：

- 新项目使用 v2
- 旧项目可迁移到 v2

---

## 最佳实践

### 1. 版本控制

```bash
# 不要提交生成的代码到版本控制
# .gitignore
packages/sdk/js/src/v2/gen/

# 只提交配置和手写代码
packages/sdk/js/src/client.ts
packages/sdk/js/src/server.ts
packages/sdk/js/script/build.ts
```

### 2. CI/CD 集成

```yaml
# .github/workflows/sdk-update.yml
name: Update SDK

on:
  push:
    paths:
      - "packages/opencode/src/api/**"

jobs:
  update-sdk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Install dependencies
        run: bun install

      - name: Generate SDK
        run: cd packages/sdk/js && bun run build

      - name: Type check
        run: cd packages/sdk/js && bun tsc

      - name: Commit changes
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add packages/sdk/js/src/v2/gen/
          git commit -m "chore: auto-update SDK"
          git push
```

### 3. 文档生成

```bash
# 从 OpenAPI 规范生成文档
npm install -g @redocly/cli
redocly build-docs openapi.json -o docs/api.html
```

### 4. 类型导出

```typescript
// packages/sdk/js/src/index.ts
// 导出生成的类型
export type * from "./gen/types.gen.js"

// 导出 SDK 类
export { OpencodeClient } from "./gen/sdk.gen.js"

// 导出便捷函数
export { createOpencodeClient } from "./client.js"
export { createOpencodeServer } from "./server.js"
export { createOpencode } from "./index.js"
```

### 5. 错误处理

```typescript
// src/error-handler.ts
import { createOpencodeClient } from "@opencode-ai/sdk"

const client = createOpencodeClient()

export async function safeApiCall<T>(fn: () => Promise<{ data?: T; error?: { detail: string } }>): Promise<T | null> {
  try {
    const result = await fn()
    if (result.error) {
      console.error("API Error:", result.error.detail)
      return null
    }
    return result.data ?? null
  } catch (error) {
    console.error("Unexpected error:", error)
    return null
  }
}

// 使用
const session = await safeApiCall(() => client.session.get({ path: { id: "123" } }))
```

---

## 总结

### 关键要点

1. **自动化生成** = 工具根据 OpenAPI 规范创建代码
2. **优势** = 效率提升 20-100 倍，准确性 100%
3. **流程** = OpenAPI → @hey-api/openapi-ts → SDK 代码
4. **维护** = 后端变更 → 重新生成

### 工作流程

```
开发流程：
1. 后端开发 API
2. 更新 OpenAPI 规范
3. 运行 bun run build
4. 测试验证
5. 发布 SDK

维护流程：
1. 后端修改 API
2. 更新 OpenAPI 规范
3. 运行 bun run build
4. 测试验证
5. 发布新版本
```

### 效果对比

| 对比项   | 手动编写 | 自动生成 |
| -------- | -------- | -------- |
| 开发时间 | 1-2 天   | 10 分钟  |
| 维护成本 | 持续高   | 极低     |
| 准确性   | 容易出错 | 100%     |
| 人力投入 | 1-2 人   | 自动化   |

---

## 参考资源

- [OpenAPI 规范文档](https://swagger.io/specification/)
- [@hey-api/openapi-ts 文档](https://heyapi.vercel.app/)
- [OpenCode SDK 源码](../packages/sdk/js)
- [OpenAPI 验证工具](https://apitools.dev/swagger-parser/online/)
