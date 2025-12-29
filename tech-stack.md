# 技术选型文档

## 项目概述

OpenCode 是一个开源的 AI 编程助手，采用现代化的技术栈构建，支持 CLI、TUI（终端用户界面）、Web 和桌面应用。项目采用 Monorepo 架构，使用 Turborepo 进行多包管理。

## 核心技术栈

### 运行时与包管理

| 技术        | 版本  | 用途                                  |
| ----------- | ----- | ------------------------------------- |
| **Bun**     | 1.3.5 | JavaScript 运行时、包管理器、构建工具 |
| **Node.js** | 22+   | 部分功能运行时（Console App 要求）    |

**选择理由**：

- Bun 提供了卓越的性能，比传统 Node.js 快 3-4 倍
- 内置的包管理器、测试运行器和打包工具，简化开发流程
- 与 Node.js 生态系统完全兼容

### 编程语言

| 技术           | 版本   | 用途                       |
| -------------- | ------ | -------------------------- |
| **TypeScript** | 5.8.2  | 主要开发语言，提供类型安全 |
| **TypeScript** | ~5.6.2 | Desktop 应用（特殊需求）   |
| **JavaScript** | -      | 部分脚本和配置             |

**选择理由**：

- TypeScript 提供静态类型检查，减少运行时错误
- 优秀的 IDE 支持和自动补全
- 更好的代码可维护性和团队协作

## 前端技术栈

### Web 框架

| 技术           | 版本                 | 用途               |
| -------------- | -------------------- | ------------------ |
| **SolidJS**    | 1.9.10               | 核心前端框架       |
| **SolidStart** | dfb2020 (pkg.pr.new) | SolidJS 的全栈框架 |

**选择理由**：

- SolidJS 具有卓越的性能，采用细粒度响应式系统
- 小巧的包体积和快速的渲染速度
- 类似 React 的开发体验，更简洁的 API

### UI 组件库

| 技术                | 版本         | 用途             |
| ------------------- | ------------ | ---------------- |
| **Kobalte**         | 0.13.11      | 无障碍 UI 组件库 |
| **@opencode-ai/ui** | workspace:\* | 自定义 UI 组件库 |

**选择理由**：

- Kobalte 提供无障碍、可自定义的组件
- 基于 SolidJS 构建，性能优异
- 支持深色模式和主题切换

### 样式方案

| 技术                  | 版本   | 用途                  |
| --------------------- | ------ | --------------------- |
| **TailwindCSS**       | 4.1.11 | 原子化 CSS 框架       |
| **@tailwindcss/vite** | 4.1.11 | Tailwind 的 Vite 集成 |

**选择理由**：

- TailwindCSS 提供快速的开发体验
- 避免编写大量 CSS 文件
- 支持深色模式和响应式设计

### 其他前端库

| 技术                                  | 版本   | 用途                           |
| ------------------------------------- | ------ | ------------------------------ |
| **Virtua**                            | 0.42.3 | 虚拟滚动库                     |
| **Solid Router**                      | 0.15.4 | 客户端路由                     |
| **Solid Primitives**                  | 4.3.3  | SolidJS 生态的原始值和实用工具 |
| **@solid-primitives/event-bus**       | 1.1.2  | 事件总线                       |
| **@solid-primitives/active-element**  | 2.1.3  | 活动元素跟踪                   |
| **@solid-primitives/audio**           | 1.4.2  | 音频处理                       |
| **@solid-primitives/bounds**          | 0.1.3  | 边界计算                       |
| **@solid-primitives/media**           | 2.3.3  | 媒体查询                       |
| **@solid-primitives/resize-observer** | 2.1.3  | 尺寸变化监听                   |
| **@solid-primitives/scroll**          | 2.1.3  | 滚动处理                       |
| **@solid-primitives/websocket**       | 1.3.1  | WebSocket 连接                 |
| **@thisbeyond/solid-dnd**             | 0.7.5  | 拖放功能                       |
| **solid-list**                        | 0.3.0  | 列表组件                       |
| **ghostty-web**                       | 0.3.0  | Web 终端模拟器                 |
| **@shikijs/transformers**             | 3.9.2  | Shiki 语法高亮转换器           |
| **Chart.js**                          | 4.5.1  | 图表库                         |

## 后端技术栈

### API 框架

| 技术             | 版本   | 用途             |
| ---------------- | ------ | ---------------- |
| **Hono**         | 4.10.7 | 高性能 Web 框架  |
| **Hono OpenAPI** | 1.1.2  | OpenAPI 规范生成 |

**选择理由**：

- Hono 极其轻量且快速，适合 Cloudflare Workers
- 简洁的 API 设计
- 内置 TypeScript 支持

### 基础设施即代码

| 技术    | 版本    | 用途             |
| ------- | ------- | ---------------- |
| **SST** | 3.17.23 | 全栈应用部署平台 |

**选择理由**：

- SST 简化了 AWS 和 Cloudflare 的部署流程
- 支持 TypeScript 配置
- 本地开发环境模拟云环境

### 部署平台

| 技术                   | 用途                |
| ---------------------- | ------------------- |
| **Cloudflare Workers** | Serverless 函数部署 |
| **Cloudflare Pages**   | 静态站点托管        |

### 云服务

| 技术                    | 版本    | 用途         |
| ----------------------- | ------- | ------------ |
| **@aws-sdk/client-s3**  | 3.933.0 | AWS S3 服务  |
| **@aws-sdk/client-sts** | 3.782.0 | AWS STS 服务 |
| **aws4fetch**           | 1.0.20  | AWS 签名请求 |

**选择理由**：

- 全球边缘网络，低延迟
- 免费额度充足
- 优秀的开发者体验

### 数据库

| 技术            | 版本   | 用途                  |
| --------------- | ------ | --------------------- |
| **PlanetScale** | 1.19.0 | MySQL 兼容数据库      |
| **Drizzle**     | 0.41.0 | TypeScript ORM        |
| **Drizzle Kit** | 0.30.5 | 数据库迁移和 CLI 工具 |
| **Postgres**    | 3.4.7  | PostgreSQL 客户端     |
| **MySQL2**      | 3.14.4 | MySQL 客户端          |

**选择理由**：

- Serverless 架构，自动扩展
- 兼容 MySQL 协议
- 支持数据库分支和回滚

### 支付集成

| 技术       | 版本   | 用途           |
| ---------- | ------ | -------------- |
| **Stripe** | 18.0.0 | 订阅和支付处理 |

## AI 相关技术

### AI SDK

| 技术                       | 版本   | 用途               |
| -------------------------- | ------ | ------------------ |
| **Vercel AI SDK**          | 5.0.97 | 统一的 AI 模型接口 |
| **@ai-sdk/provider**       | 2.0.0  | AI 提供商基础接口  |
| **@ai-sdk/provider-utils** | 3.0.18 | AI 提供商工具函数  |

**选择理由**：

- 提供统一的 API 接口
- 支持流式响应
- 内置工具调用支持

### AI 提供商支持

| 技术包                      | 版本   | 支持的提供商           | 使用的包                     |
| --------------------------- | ------ | ---------------------- | ---------------------------- |
| @ai-sdk/anthropic           | 2.0.50 | Claude                 | opencode, console-function\* |
| @ai-sdk/openai              | 2.0.71 | OpenAI GPT             | opencode, console-function\* |
| @ai-sdk/google              | 2.0.44 | Google Gemini          | opencode                     |
| @ai-sdk/google-vertex       | 3.0.81 | Google Vertex AI       | opencode                     |
| @ai-sdk/amazon-bedrock      | 3.0.57 | Amazon Bedrock         | opencode                     |
| @ai-sdk/azure               | 2.0.73 | Azure OpenAI           | opencode                     |
| @ai-sdk/groq                | 2.0.33 | Groq                   | opencode                     |
| @ai-sdk/cohere              | 2.0.21 | Cohere                 | opencode                     |
| @ai-sdk/mistral             | 2.0.26 | Mistral AI             | opencode                     |
| @ai-sdk/togetherai          | 1.0.30 | Together AI            | opencode                     |
| @ai-sdk/perplexity          | 2.0.22 | Perplexity             | opencode                     |
| @ai-sdk/xai                 | 2.0.42 | xAI                    | opencode                     |
| @ai-sdk/cerebras            | 1.0.33 | Cerebras               | opencode                     |
| @ai-sdk/deepinfra           | 1.0.30 | DeepInfra              | opencode                     |
| @ai-sdk/openai-compatible   | 1.0.27 | OpenAI 兼容 API        | opencode, console-function\* |
| @ai-sdk/mcp                 | 0.0.8  | Model Context Protocol | opencode                     |
| @ai-sdk/gateway             | 2.0.23 | Vercel AI SDK Gateway  | opencode                     |
| @openrouter/ai-sdk-provider | 1.5.2  | OpenRouter             | opencode                     |

\*console-function 包使用旧版本（anthropic: 2.0.0, openai: 2.0.2, openai-compatible: 1.0.1）

**选择理由**：

- 支持多种 AI 提供商，避免 vendor lock-in
- 用户可以根据需求和预算选择不同的模型
- 统一的 API 接口降低切换成本

### 协议支持

| 技术                           | 版本   | 用途         |
| ------------------------------ | ------ | ------------ |
| **Model Context Protocol SDK** | 1.15.1 | MCP 协议实现 |
| **Agent Client Protocol SDK**  | 0.5.1  | ACP 协议实现 |

**选择理由**：

- MCP 和 ACP 是 AI 智能体领域的开放标准
- 支持与其他 MCP/ACP 工具互操作
- 推动生态系统发展

## 桌面应用

### 跨平台框架

| 技术      | 版本 | 用途               |
| --------- | ---- | ------------------ |
| **Tauri** | 2.x  | 跨平台桌面应用框架 |

**选择理由**：

- Tauri 使用 Web 技术构建 UI，后端使用 Rust
- 比 Electron 更轻量、更安全
- 更小的安装包体积和更低的内存占用

### 支持平台

| 平台                  | 打包格式             |
| --------------------- | -------------------- |
| macOS (Apple Silicon) | .dmg                 |
| macOS (Intel)         | .dmg                 |
| Windows               | .exe (NSIS)          |
| Linux                 | .deb, .rpm, AppImage |

## 文档站点

### 文档框架

| 技术          | 版本   | 用途             |
| ------------- | ------ | ---------------- |
| **Astro**     | 5.7.13 | 静态站点生成器   |
| **Starlight** | 0.34.3 | Astro 的文档主题 |

**选择理由**：

- Astro 提供极快的页面加载速度
- Starlight 提供开箱即用的文档功能
- 支持深色模式、搜索、多语言等功能

### Markdown 渲染

| 技术                         | 版本   | 用途                |
| ---------------------------- | ------ | ------------------- |
| **Marked**                   | 17.0.1 | Markdown 解析器     |
| **Marked Shiki**             | 1.2.1  | 代码语法高亮        |
| **Shiki**                    | 3.20.0 | 语法高亮引擎        |
| **@shikijs/transformers**    | 3.4.2  | Shiki 转换器（web） |
| **rehype-autolink-headings** | 7.1.0  | Markdown 增强插件   |

### Astro 集成

| 技术                         | 版本   | 用途                    |
| ---------------------------- | ------ | ----------------------- |
| **@astrojs/cloudflare**      | 12.6.3 | Cloudflare 集成         |
| **@astrojs/markdown-remark** | 6.3.1  | Markdown 和 Remark 集成 |
| **@astrojs/solid-js**        | 5.1.0  | SolidJS 集成            |

### 文档主题和字体

| 技术                          | 版本  | 用途               |
| ----------------------------- | ----- | ------------------ |
| **toolbeam-docs-theme**       | 0.4.8 | 自定义文档主题     |
| **@fontsource/ibm-plex-mono** | 5.2.5 | IBM Plex Mono 字体 |
| **@ibm/plex**                 | 6.4.1 | IBM Plex 字体家族  |

## 终端用户界面 (TUI)

### TUI 框架

| 技术               | 版本   | 用途         |
| ------------------ | ------ | ------------ |
| **@opentui/core**  | 0.1.63 | TUI 核心框架 |
| **@opentui/solid** | 0.1.63 | SolidJS 集成 |

**选择理由**：

- 基于现代 Web 技术构建 TUI
- 与前端框架（SolidJS）共享代码逻辑
- 提供流畅的终端交互体验

### 代码分析

| 技术                 | 版本    | 用途                     |
| -------------------- | ------- | ------------------------ |
| **Tree-sitter**      | 0.25.0  | 代码解析器               |
| **Web Tree-sitter**  | 0.25.10 | Tree-sitter 的 WASM 版本 |
| **Tree-sitter-bash** | 0.25.0  | Bash 语言支持            |

**选择理由**：

- Tree-sitter 提供增量解析，性能优异
- 支持错误恢复和语法树构建
- 为 LSP（Language Server Protocol）提供基础

### LSP 支持

| 技术                            | 版本   | 用途              |
| ------------------------------- | ------ | ----------------- |
| **vscode-languageserver-types** | 3.17.5 | LSP 类型定义      |
| **vscode-jsonrpc**              | 8.2.1  | JSON-RPC 协议实现 |

**选择理由**：

- LSP 是语言服务器协议的标准
- 支持与多种语言服务器通信
- 提供代码补全、跳转定义等功能

## 工具库

### 数据校验

| 技术                      | 版本  | 用途                          |
| ------------------------- | ----- | ----------------------------- |
| **Zod**                   | 4.1.8 | TypeScript-first 的数据校验库 |
| **@standard-schema/spec** | 1.0.0 | 标准化 Schema 规范            |

**选择理由**：

- Zod 提供类型推导，与 TypeScript 无缝集成
- 支持复杂的 schema 定义
- 可以生成 JSON Schema

### 工具函数

| 技术       | 版本   | 用途         |
| ---------- | ------ | ------------ |
| **Remeda** | 2.26.0 | 函数式工具库 |

**选择理由**：

- Remeda 是一个轻量级的函数式工具库
- 比 Lodash 更现代，TypeScript 支持更好
- Tree-shakable，减小打包体积

### 日期时间

| 技术      | 版本  | 用途           |
| --------- | ----- | -------------- |
| **Luxon** | 3.6.1 | 日期时间处理库 |

**选择理由**：

- Luxon 提供不可变的日期时间对象
- 时区支持优秀
- API 设计直观

### 文本比较

| 技术              | 版本  | 用途         |
| ----------------- | ----- | ------------ |
| **Diff**          | 8.0.2 | 文本差异比较 |
| **@pierre/diffs** | 1.0.2 | 差异增强     |

### 数据处理

| 技术             | 版本   | 用途              |
| ---------------- | ------ | ----------------- |
| **decimal.js**   | 10.5.0 | 高精度十进制运算  |
| **gray-matter**  | 4.0.3  | Front Matter 解析 |
| **partial-json** | 0.1.7  | 部分 JSON 解析    |
| **js-base64**    | 3.7.7  | Base64 编解码     |
| **lang-map**     | 0.4.0  | 语言扩展名映射    |

**选择理由**：

- decimal.js 提供精确的十进制运算，避免浮点数精度问题
- gray-matter 用于解析 Markdown 文件的 Front Matter
- partial-json 支持流式解析不完整的 JSON

### ID 生成

| 技术     | 版本  | 用途           |
| -------- | ----- | -------------- |
| **ULID** | 3.0.1 | 唯一标识符生成 |

### 终端相关

| 技术           | 版本   | 用途             |
| -------------- | ------ | ---------------- |
| **bun-pty**    | 0.4.2  | Bun 伪终端支持   |
| **clipboardy** | 4.0.0  | 跨平台剪贴板操作 |
| **yargs**      | 18.0.0 | 命令行参数解析   |

### 文本处理

| 技术             | 版本   | 用途              |
| ---------------- | ------ | ----------------- |
| **jsonc-parser** | 3.3.1  | JSONC 解析        |
| **minimatch**    | 10.0.3 | 模式匹配          |
| **strip-ansi**   | 7.1.2  | ANSI 转义序列移除 |
| **turndown**     | 7.2.0  | HTML 转 Markdown  |

### 邮件

| 技术          | 版本  | 用途         |
| ------------- | ----- | ------------ |
| **JSX Email** | 1.1.1 | 邮件模板渲染 |

### AI 客户端

| 技术       | 版本   | 用途            |
| ---------- | ------ | --------------- |
| **OpenAI** | 5.11.0 | OpenAI SDK 备用 |

### 其他工具

| 技术               | 版本   | 用途              |
| ------------------ | ------ | ----------------- |
| **open**           | 10.1.2 | 打开文件/URL/应用 |
| **@zip.js/zip.js** | 2.7.62 | ZIP 文件处理      |

**选择理由**：

- ULID 是可排序的唯一标识符
- 比 UUID 更适合分布式系统
- 包含时间戳，便于排序

## 构建与开发工具

### 构建工具

| 技术                        | 版本          | 用途                     |
| --------------------------- | ------------- | ------------------------ |
| **Vite**                    | 7.1.4         | 前端构建工具             |
| **Turbo**                   | 2.5.6         | Monorepo 构建系统        |
| **Esbuild**                 | -             | 快速的 JavaScript 打包器 |
| **Nitro**                   | 3.0.1-alpha.1 | 通用服务器引擎           |
| **@cloudflare/vite-plugin** | 1.15.2        | Cloudflare Vite 插件     |
| **Wrangler**                | 4.50.0        | Cloudflare Workers CLI   |

**选择理由**：

- Vite 提供极快的开发服务器启动和 HMR
- Turbo 优化 Monorepo 的构建性能
- Esbuild 作为底层打包器，提供极高的编译速度

### 代码质量

| 技术         | 版本  | 用途           |
| ------------ | ----- | -------------- |
| **Prettier** | 3.6.2 | 代码格式化     |
| **Husky**    | 9.1.7 | Git hooks 管理 |

**选择理由**：

- Prettier 自动格式化代码，保持一致的代码风格
- Husky 在 Git 钩子中运行 lint 和测试，确保代码质量

### 文件监听

| 技术                | 版本  | 用途             |
| ------------------- | ----- | ---------------- |
| **@parcel/watcher** | 2.5.1 | 跨平台文件监听   |
| **Chokidar**        | 4.0.3 | Node.js 文件监听 |

**选择理由**：

- @parcel/watcher 使用原生实现，性能更好
- Chokidar 作为备用方案

## 集成与部署

### 持续集成

| 技术               | 用途         |
| ------------------ | ------------ |
| **GitHub Actions** | CI/CD 流水线 |

**选择理由**：

- GitHub Actions 与 GitHub 仓库深度集成
- 免费额度充足
- 丰富的第三方 Actions 市场

### 发布

| 技术                | 用途         |
| ------------------- | ------------ |
| **NPM**             | 包发布       |
| **GitHub Releases** | 桌面应用发布 |

## 第三方集成

### GitHub

| 技术                        | 版本   | 用途                      |
| --------------------------- | ------ | ------------------------- |
| **@octokit/rest**           | 22.0.0 | GitHub REST API 客户端    |
| **@octokit/graphql**        | 9.0.2  | GitHub GraphQL API 客户端 |
| **@octokit/webhooks-types** | 7.6.1  | GitHub Webhook 类型       |

### 认证

| 技术                     | 版本                 | 用途       |
| ------------------------ | -------------------- | ---------- |
| **@openauthjs/openauth** | 0.0.0-20250322224806 | 认证和授权 |
| **Jose**                 | 6.0.11               | JWT 处理   |

### 其他

| 技术                  | 版本                | 用途                         |
| --------------------- | ------------------- | ---------------------------- |
| **@actions/core**     | 1.11.1              | GitHub Actions 核心          |
| **@actions/github**   | 6.0.1               | GitHub Actions GitHub 上下文 |
| **@actions/artifact** | 5.0.1 (desktop: 4.0.0) | GitHub Actions 工件管理      |

## 项目架构

### Monorepo 结构

```
opencode/
├── packages/
│   ├── opencode/          # 核心 CLI 和 TUI 应用
│   ├── console/           # Web 控制台
│   │   ├── app/          # SolidJS 前端
│   │   ├── core/         # 核心逻辑
│   │   ├── function/     # Cloudflare Workers 函数
│   │   ├── mail/         # 邮件模板
│   │   └── resource/     # 资源配置
│   ├── desktop/           # Tauri 桌面应用
│   ├── web/               # Astro 文档站点
│   ├── sdk/               # JavaScript SDK
│   ├── ui/                # UI 组件库
│   ├── util/              # 工具函数库
│   ├── script/            # 共享脚本
│   ├── plugin/            # 插件系统
│   ├── function/          # 通用 Cloudflare Workers
│   ├── enterprise/        # 企业版功能
│   ├── slack/             # Slack 集成
│   └── app/               # 通用应用代码
├── sdks/
│   └── vscode/            # VS Code 扩展
├── infra/                 # SST 基础设施配置
└── github/                # GitHub 相关脚本
```

### 工作区管理

使用 Bun Workspaces 和 Turborepo 进行 Monorepo 管理：

- **Bun Workspaces**: 管理包之间的依赖关系
- **Turborepo**: 优化构建和任务执行性能

## 版本说明

### Bun 版本

- packageManager 声明：1.3.5
- @types/bun（catalog）：1.3.4
- @types/bun（console/core）：1.3.0

### TypeScript 版本

- 主版本（catalog）：5.8.2
- Desktop 应用：~5.6.2（特殊需求）

### AI SDK 版本差异

不同包使用不同版本的 AI SDK：

- **opencode 包**：使用最新版本（如 @ai-sdk/anthropic 2.0.50）
- **console-function 包**：使用旧版本（如 @ai-sdk/anthropic 2.0.0）

这是正常的版本演进过程，新功能优先在核心包（opencode）中使用。

### GitHub Actions 工具版本差异

- **@actions/artifact**：desktop 包使用 4.0.0，其他包使用 5.0.1

这是为了保持桌面应用构建流程的稳定性。

### Catalog 依赖管理

项目使用 Bun Workspaces 的 catalog 功能统一管理依赖版本，catalog: 引用表示使用根 package.json 中定义的版本。

## 技术选型总结

### 核心原则

1. **性能优先**: 选择 Bun、SolidJS、Hono 等高性能技术栈
2. **开发体验**: TypeScript、Vite、SST 提供优秀的开发体验
3. **可扩展性**: Monorepo 架构、插件系统、SDK 支持
4. **开放性**: 支持 AI 模型无关设计，避免 vendor lock-in
5. **现代化**: 使用最新、最活跃的开源技术

### 技术亮点

- **全 TypeScript**: 类型安全贯穿整个项目
- **多平台支持**: CLI、TUI、Web、Desktop 全覆盖
- **AI 模型无关**: 支持多种 AI 提供商，灵活切换
- **LSP 原生支持**: 深度集成语言服务器协议
- **Serverless 架构**: 基于 Cloudflare Workers 的无服务器部署
- **现代化构建**: 使用 Bun、Vite、Turbo 提供极快的构建速度

### 技术债务与改进方向

1. **Tauri 桌面应用**: 仍处于 BETA 阶段，需持续优化
2. **部分依赖版本**: 使用 pkg.pr.new 等预发布版本，需关注稳定性
3. **测试覆盖**: 需要补充更多的单元测试和集成测试
4. **文档完善**: 部分 API 文档需要补充详细说明
5. **版本一致性**:
   - TypeScript 版本在不同包中不一致（desktop 使用 ~5.6.2，其他使用 5.8.2）
   - AI SDK 版本在不同包中不一致（console-function 使用旧版本）
   - @types/bun 版本不一致（console/core 使用 1.3.0，catalog 为 1.3.4）
   - @actions/artifact 版本不一致（desktop 使用 4.0.0，其他使用 5.0.1）
6. **依赖更新**: 部分 AI SDK 和 Bun types 需要更新到最新稳定版本

## 相关资源

- [项目主页](https://opencode.ai)
- [GitHub 仓库](https://github.com/sst/opencode)
- [文档](https://opencode.ai/docs)
- [Discord 社区](https://opencode.ai/discord)

---

最后更新: 2025-12-29
