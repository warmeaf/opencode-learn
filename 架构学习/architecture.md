# OpenCode 项目架构文档

## 项目概述

OpenCode 是一个开源的 AI 编程代理项目，提供 CLI、Web、桌面应用等多种使用方式。项目采用 Monorepo 架构，支持多种 AI 模型提供商，具有完整的插件系统和企业级功能。

## 目录结构

```
opencode/
├── packages/              # 核心代码包（monorepo）
│   ├── opencode/         # CLI 核心包
│   ├── web/              # Web 文档站点
│   ├── app/              # Web 应用
│   ├── desktop/          # 桌面应用（Tauri）
│   ├── ui/               # 共享 UI 组件库
│   ├── sdk/              # JavaScript/TypeScript SDK
│   │   └── js/           # SDK 实现
│   ├── plugin/           # 插件系统
│   ├── util/             # 工具函数库
│   ├── script/           # 脚本工具
│   ├── console/          # 管理控制台
│   │   ├── app/          # 控制台前端
│   │   ├── core/         # 核心业务逻辑
│   │   ├── function/     # 云函数
│   │   ├── mail/         # 邮件服务
│   │   └── resource/     # 资源管理
│   ├── function/         # 云函数
│   ├── enterprise/       # 企业版
│   ├── slack/            # Slack 集成
│   ├── docs/             # 文档资源
│   ├── extensions/       # 扩展（如 Zed）
│   └── identity/         # 身份认证
├── infra/                # 基础设施代码（SST）
└── specs/                # 规范文档
```

## 核心架构

### Monorepo 架构

- **包管理器**: Bun 1.3.5
- **工作区工具**: Turborepo
- **部署平台**: SST + Cloudflare

### 主要子系统

#### 1. CLI 核心 (`packages/opencode/`)
- **入口**: `src/index.ts`
- **功能**:
  - CLI 命令行接口
  - AI 代理系统
  - TUI（终端用户界面）
  - 服务器/客户端架构
  - MCP（Model Context Protocol）集成
  - LSP（Language Server Protocol）支持

#### 2. 前端应用
- **Web 应用** (`packages/app/`): SolidJS + Vite
- **桌面应用** (`packages/desktop/`): Tauri 2.x
- **UI 组件库** (`packages/ui/`): 共享的 SolidJS 组件
- **Web 文档** (`packages/web/`): Astro + Starlight

#### 3. SDK & 插件
- **SDK** (`packages/sdk/js/`): JavaScript/TypeScript SDK
- **插件系统** (`packages/plugin/`): 插件开发框架
- **工具库** (`packages/util/`): 共享工具函数

#### 4. 云服务
- **管理控制台** (`packages/console/`):
  - 前端应用
  - 核心业务逻辑（数据库、Stripe 集成）
  - 邮件服务
  - 资源管理（Cloudflare/Node.js）
- **云函数** (`packages/function/`): GitHub 集成等
- **企业版** (`packages/enterprise/`): 企业级功能

#### 5. 集成
- **Slack 集成** (`packages/slack/`): Slack Bot
- **GitHub 集成**: PR 管理、Actions 集成

## 技术栈

### 开发语言与运行时
- **TypeScript 5.8.2**: 主要开发语言
- **Bun**: 运行时和包管理器
- **Node.js 22+**: 兼容性支持

### 前端框架
- **SolidJS 1.9.10**: 主要前端框架
- **Astro 5.7.13**: 文档站点生成器
- **Starlight 0.34.3**: Astro 文档主题
- **TailwindCSS 4.1.11**: 样式框架
- **Kobalte**: SolidJS UI 组件库
- **Vite 7.1.4**: 构建工具

### 桌面应用
- **Tauri 2.x**: 跨平台桌面应用框架
- **Ghostty Web**: 终端模拟器

### 后端与部署
- **SST 3.17.23**: Serverless 部署框架
- **Cloudflare Workers**: 边缘计算平台
- **Hono 4.10.7**: 轻量级 Web 框架
- **Drizzle ORM**: 数据库 ORM
- **PlanetScale**: 数据库服务
- **Stripe**: 支付处理

### AI 集成
- **Vercel AI SDK 5.0.97**: AI 模型抽象层
- **多提供商支持**: Anthropic、OpenAI、Google、Azure、Groq、Mistral、Cohere 等
- **MCP SDK**: Model Context Protocol 支持

### 开发工具
- **Turbo 2.5.6**: Monorepo 构建系统
- **Prettier 3.6.2**: 代码格式化
- **Husky 9.1.7**: Git hooks
- **Tree-sitter**: 代码解析

## 架构特点

1. **客户端/服务器分离**: CLI 可作为服务器运行，支持远程客户端连接
2. **多端支持**: CLI、TUI、Web、桌面应用、Slack Bot
3. **模块化设计**: 清晰的包边界和职责划分
4. **插件系统**: 支持扩展功能
5. **多云部署**: 支持 Cloudflare 等多种部署目标
6. **AI 模型无关性**: 支持多个 AI 提供商
7. **企业级功能**: 包含完整的控制台、支付、用户管理等
