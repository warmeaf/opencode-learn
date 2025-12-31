# TypeScript 命名空间

## 什么是 TypeScript 命名空间

TypeScript 命名空间是一种将相关代码组织在一起的方式，使用 `namespace` 关键字声明。它可以将类型、函数、变量等逻辑分组，避免全局命名冲突。

**基本语法：**

```typescript
export namespace Config {
  const Info = z.object({...})
  export type Info = z.infer<typeof Info>
  export async function get() { ... }
}
```

**使用方式：**

```typescript
const config = await Config.get() // Config.get()
```

## 为什么这个项目需要命名空间

### 1. 模块化组织

将相关功能分组：`File`、`Session`、`Config`、`Log` 等，每个命名空间代表一个功能域。

### 2. 类型和实现统一管理

在同一命名空间中定义 Zod schema 和对应的实现函数，如 `packages/opencode/src/config/config.ts:22-1024` 中的 `Config.Info` 和 `Config.get()`。

### 3. 避免命名冲突

不同模块可能有同名函数（如 `read`），命名空间隔离它们：

- `File.read()`
- `Session.read()`

### 4. 清晰的 API 语义

使用 `File.list()`、`Session.create()` 比单纯的 `listFiles()`、`createSession()` 更明确来源。

### 5. 便于导出和引用

通过 `export namespace` 将一组相关 API 统一导出，如 `packages/opencode/src/file/index.ts:16` 的 `File` 命名空间包含 `Info`、`Node`、`Content` 等类型和 `read()`、`list()` 等函数。

## 项目中的命名空间使用示例

### File 命名空间

位置：`packages/opencode/src/file/index.ts:16`

```typescript
export namespace File {
  const log = Log.create({ service: "file" })

  export const Info = z.object({
    path: z.string(),
    added: z.number().int(),
    removed: z.number().int(),
    status: z.enum(["added", "deleted", "modified"]),
  })

  export type Info = z.infer<typeof Info>

  export async function read(file: string): Promise<Content> { ... }
  export async function list(dir?: string) { ... }
  export async function search(input: { query: string; limit?: number; dirs?: boolean; type?: "file" | "directory" }) { ... }
}
```

### Session 命名空间

位置：`packages/opencode/src/session/index.ts:22`

```typescript
export namespace Session {
  const log = Log.create({ service: "session" })

  export const Info = z.object({
    id: Identifier.schema("session"),
    projectID: z.string(),
    directory: z.string(),
    parentID: Identifier.schema("session").optional(),
    summary: z.object({...}).optional(),
    share: z.object({...}).optional(),
    title: z.string(),
    version: z.string(),
    time: z.object({...}),
    revert: z.object({...}).optional(),
  })

  export type Info = z.output<typeof Info>

  export const create = fn(...)
  export const fork = fn(...)
  export async function get(id: string) { ... }
  export async function* list() { ... }
}
```

### Config 命名空间

位置：`packages/opencode/src/config/config.ts:22`

```typescript
export namespace Config {
  const log = Log.create({ service: "config" })

  export const Info = z.object({
    $schema: z.string().optional(),
    theme: z.string().optional(),
    keybinds: Keybinds.optional(),
    logLevel: Log.Level.optional(),
    tui: TUI.optional(),
    // ... 更多配置字段
  })

  export type Info = z.output<typeof Info>

  export async function get() {
    return state().then((x) => x.config)
  }

  export async function update(config: Info) { ... }
}
```

### Log 命名空间

位置：`packages/opencode/src/util/log.ts:6`

```typescript
export namespace Log {
  export const Level = z.enum(["DEBUG", "INFO", "WARN", "ERROR"])
  export type Level = z.infer<typeof Level>

  export async function init(options: Options) { ... }
  export function create(tags?: Record<string, any>) { ... }
}
```

### Rpc 命名空间

位置：`packages/opencode/src/util/rpc.ts:1`

```typescript
export namespace Rpc {
  type Definition = {
    [method: string]: (input: any) => any
  }

  export function listen(rpc: Definition) { ... }
  export function client<T extends Definition>(target: { ... }) { ... }
}
```

## 命名空间 vs 模块

在这个项目中，命名空间与 ES6 模块配合使用：

- **模块**：用于文件级别的导入/导出（`import/export`）
- **命名空间**：用于文件内部的逻辑分组，组织相关的类型和函数

这种组合方式的优势：

1. 保持了模块化（每个文件是一个独立的模块）
2. 在文件内部提供清晰的逻辑分组
3. API 使用时具有明确的语义（`File.read()` 而不是 `read()`）
4. 类型推导更加清晰（`File.Info`、`Session.Info`）

## 命名空间的使用模式

### 模式 1：类型 + 实现

将类型定义和实现函数放在同一命名空间中：

```typescript
export namespace SomeModule {
  export const Schema = z.object({...})
  export type Schema = z.infer<typeof Schema>

  export async function get(): Promise<Schema> { ... }
  export async function update(data: Schema) { ... }
}
```

### 模式 2：常量 + 工具函数

将相关常量和工具函数分组：

```typescript
export namespace Util {
  const config = {...}

  export function helper1() { ... }
  export function helper2() { ... }
}
```

### 模式 3：事件定义

将与事件相关的定义分组：

```typescript
export namespace Module {
  export const Event = {
    Created: BusEvent.define("module.created", z.object({...})),
    Updated: BusEvent.define("module.updated", z.object({...})),
    Deleted: BusEvent.define("module.deleted", z.object({...})),
  }
}
```

## 现代 TypeScript 开发中的命名空间使用

### 项目现状统计

通过代码扫描，该项目共使用了 **79 处** `namespace`：

```bash
grep -r "namespace\s+\w+" packages/opencode/src
# 79 matches across multiple modules
```

主要使用的命名空间包括：

- `Log` - 日志管理
- `File` - 文件操作
- `Config` - 配置管理
- `Session` - 会话管理
- `Provider` - 提供商配置
- `Agent` - 代理配置
- `Tool`、`Storage`、`Plugin` 等核心模块

### 与现代最佳实践的对比

**现代 TypeScript 开发建议：**

除非有特殊需求（如全局类型、兼容旧代码），否则更推荐**直接导出函数/对象/类型**，而非使用命名空间。

**当前项目的问题分析：**

#### 1. 模块内的冗余性

在 ES6 模块中使用 `namespace` 是冗余的：

```typescript
// 当前写法 (packages/opencode/src/util/log.ts:6)
export namespace Log {
  export const Level = z.enum(["DEBUG", "INFO", "WARN", "ERROR"])
  export type Level = z.infer<typeof Level>
  export function create(tags?: Record<string, any>) { ... }
}

// 使用时
import { Log } from "./util/log"
const log = Log.create({ service: "xxx" })
```

**推荐写法：**

```typescript
// 直接导出
export const Level = z.enum(["DEBUG", "INFO", "WARN", "ERROR"])
export type Level = z.infer<typeof Level>
export function create(tags?: Record<string, any>) { ... }

// 使用时
import { create as createLogger } from "./util/log"
const log = createLogger({ service: "xxx" })
```

#### 2. 导入复杂化

命名空间需要额外的导入和使用层级：

```typescript
// 当前
import { File } from "@/file"
const files = await File.list()

// 直接导出
import { list as listFiles } from "@/file"
const files = await listFiles()
```

#### 3. Tree-shaking 影响

虽然现代打包工具（如 esbuild、Rollup）能处理命名空间，但直接导出通常更有利于 tree-shaking。

### 项目中命名空间的合理性

尽管不符合现代最佳实践，但本项目使用命名空间有其合理性：

#### 1. 单文件内大量相关内容组织

如 `packages/opencode/src/config/config.ts` 包含 **1000+ 行**代码，命名空间提供了清晰的逻辑分组：

```typescript
export namespace Config {
  // 类型定义
  export const Info = z.object({...})
  export const Agent = z.object({...})
  export const Provider = z.object({...})

  // 实现函数
  export async function get() { ... }
  export async function update(config: Info) { ... }

  // 错误类型
  export const JsonError = NamedError.create(...)
  export const InvalidError = NamedError.create(...)
}
```

#### 2. 统一的 API 语义

所有导出统一在命名空间下，提供语义分组和来源清晰度：

```typescript
Config.get() // 明确是配置相关
File.read() // 明确是文件操作
Session.create() // 明确是会话管理
```

#### 3. 项目风格一致性

整个项目保持一致的模式，便于代码理解和维护。79 处命名空间使用形成了一致的代码风格。

#### 4. 避免命名冲突

不同模块的同名函数通过命名空间自然隔离：

```typescript
// 每个模块都有自己的 init()、get() 等
Config.init()
File.init()
Session.init()
```

### 重构建议

#### 新项目建议

对于新项目，推荐：

1. **避免使用 `namespace`**
2. **直接导出函数和类型**
3. **使用具名导出而非默认导出**
4. **通过文件结构组织模块**

```typescript
// ✅ 推荐
// file.ts
export interface Info { ... }
export function read() { ... }
export function list() { ... }

// ❌ 避免
// file.ts
export namespace File {
  export interface Info { ... }
  export function read() { ... }
}
```

#### 当前项目建议

考虑到本项目已经：

1. 形成了一致的代码风格
2. 大量依赖命名空间（79 处）
3. 重构成本极高
4. 团队已经熟悉这种模式

**建议：暂维持现状**

但可以：

1. **新功能模块考虑直接导出**，逐步过渡
2. **在文档中说明命名空间的使用约定**
3. **在团队评审时评估是否需要命名空间**

### 结论

本项目的命名空间使用虽然不符合现代 TypeScript 最佳实践，但考虑到：

- 已有的代码规模和一致性
- 单文件内大量相关内容的组织需求
- 团队的熟悉程度
- 重构成本与收益比

**暂不推荐大规模重构**。但对于新项目或独立模块，应遵循现代最佳实践，避免使用命名空间。
