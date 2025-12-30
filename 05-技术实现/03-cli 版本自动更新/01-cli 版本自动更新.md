# CLI版本自动更新机制

## 概述

OpenCode CLI 每次启动时都会自动检查版本并更新，确保用户始终使用最新版本。本文档详细解析这一机制的实现原理。

## 执行流程

### 1. 入口文件

**文件位置**: `packages/opencode/bin/opencode`

入口文件是一个Node.js脚本，负责查找并执行对应平台的二进制文件：

```javascript
const base = "opencode-" + platform + "-" + arch
const binary = platform === "windows" ? "opencode.exe" : "opencode"

function findBinary(startDir) {
  // 从当前目录向上查找 node_modules 中匹配的 opencode 包
  for (;;) {
    const modules = path.join(current, "node_modules")
    if (fs.existsSync(modules)) {
      const entries = fs.readdirSync(modules)
      for (const entry of entries) {
        if (!entry.startsWith(base)) continue
        const candidate = path.join(modules, entry, "bin", binary)
        if (fs.existsSync(candidate)) {
          return candidate
        }
      }
    }
    const parent = path.dirname(current)
    if (parent === current) return
    current = parent
  }
}
```

### 2. TUI启动时触发版本检查

**文件位置**:

- `packages/opencode/src/cli/cmd/tui/spawn.ts:16`
- `packages/opencode/src/cli/cmd/tui/worker.ts:50`

当用户使用 TUI 界面时（如直接运行 `opencode` 不带子命令），系统会启动版本检查：

```typescript
// spawn.ts
export const TuiSpawnCommand = cmd({
  command: "spawn [project]",
  handler: async (args) => {
    upgrade()  // ← 启动时调用
    // ...
  },
})

// worker.ts - 通过 RPC 远程调用检查升级
async checkUpgrade(input: { directory: string }) {
  await Instance.provide({
    directory: input.directory,
    init: InstanceBootstrap,
    fn: async () => {
      await upgrade().catch(() => {})
    },
  }),
}
```

### 3. 版本检查逻辑

**文件位置**: `packages/opencode/src/cli/upgrade.ts`

```typescript
export async function upgrade() {
  const config = await Config.global()
  const method = await Installation.method()
  const latest = await Installation.latest(method).catch(() => {})
  if (!latest) return
  if (Installation.VERSION === latest) return // 已是最新版本

  // 检查配置，决定是否自动更新
  if (config.autoupdate === false || Flag.OPENCODE_DISABLE_AUTOUPDATE) {
    return // 已禁用自动更新
  }
  if (config.autoupdate === "notify") {
    await Bus.publish(Installation.Event.UpdateAvailable, { version: latest })
    return // 仅通知不更新
  }

  if (method === "unknown") return
  // 执行自动更新
  await Installation.upgrade(method, latest)
    .then(() => Bus.publish(Installation.Event.Updated, { version: latest }))
    .catch(() => {})
}
```

### 4. 获取最新版本

**文件位置**: `packages/opencode/src/installation/index.ts:167`

根据不同的安装方式，从不同的源获取最新版本：

```typescript
export async function latest(installMethod?: Method) {
  const detectedMethod = installMethod || (await method())

  // Brew 安装方式
  if (detectedMethod === "brew") {
    const formula = await getBrewFormula()
    if (formula === "opencode") {
      return fetch("https://formulae.brew.sh/api/formula/opencode.json")
        .then((res) => res.json())
        .then((data: any) => data.versions.stable)
    }
  }

  // NPM/Bun/PNPM 安装方式
  if (detectedMethod === "npm" || detectedMethod === "bun" || detectedMethod === "pnpm") {
    const registry = await iife(async () => {
      const r = (await $`npm config get registry`.quiet().nothrow().text()).trim()
      const reg = r || "https://registry.npmjs.org"
      return reg.endsWith("/") ? reg.slice(0, -1) : reg
    })
    const channel = CHANNEL // "latest" 或其他通道
    return fetch(`${registry}/opencode-ai/${channel}`)
      .then((res) => res.json())
      .then((data: any) => data.version)
  }

  // Curl 安装方式 - 从 GitHub releases 获取
  return fetch("https://api.github.com/repos/sst/opencode/releases/latest")
    .then((res) => res.json())
    .then((data: any) => data.tag_name.replace(/^v/, ""))
}
```

### 5. 检测安装方式

**文件位置**: `packages/opencode/src/installation/index.ts:60`

```typescript
export async function method() {
  // 检查是否为 curl 安装
  if (process.execPath.includes(path.join(".opencode", "bin"))) return "curl"
  if (process.execPath.includes(path.join(".local", "bin"))) return "curl"

  const exec = process.execPath.toLowerCase()

  const checks = [
    { name: "npm" as const, command: () => $`npm list -g --depth=0` },
    { name: "yarn" as const, command: () => $`yarn global list` },
    { name: "pnpm" as const, command: () => $`pnpm list -g --depth=0` },
    { name: "bun" as const, command: () => $`bun pm ls -g` },
    { name: "brew" as const, command: () => $`brew list --formula opencode` },
  ]

  // 优先检查与当前执行路径匹配的包管理器
  checks.sort((a, b) => {
    const aMatches = exec.includes(a.name)
    const bMatches = exec.includes(b.name)
    if (aMatches && !bMatches) return -1
    if (!aMatches && bMatches) return 1
    return 0
  })

  // 执行检查命令
  for (const check of checks) {
    const output = await check.command()
    if (output.includes(check.name === "brew" ? "opencode" : "opencode-ai")) {
      return check.name
    }
  }

  return "unknown"
}
```

### 6. 执行更新

**文件位置**: `packages/opencode/src/installation/index.ts:121`

根据安装方式使用对应的包管理器更新：

```typescript
export async function upgrade(method: Method, target: string) {
  let cmd
  switch (method) {
    case "curl":
      cmd = $`curl -fsSL https://opencode.ai/install | bash`.env({
        ...process.env,
        VERSION: target, // 指定要安装的版本
      })
      break
    case "npm":
      cmd = $`npm install -g opencode-ai@${target}`
      break
    case "pnpm":
      cmd = $`pnpm install -g opencode-ai@${target}`
      break
    case "bun":
      cmd = $`bun install -g opencode-ai@${target}`
      break
    case "brew":
      const formula = await getBrewFormula()
      cmd = $`brew install ${formula}`
      break
    default:
      throw new Error(`Unknown method: ${method}`)
  }

  const result = await cmd.quiet().throws(false)
  if (result.exitCode !== 0) throw new UpgradeFailedError({ stderr: result.stderr.toString() })
}
```

## 配置选项

自动更新行为可以通过配置文件控制：

```json
{
  "autoupdate": true           // 自动更新（默认）
  "autoupdate": false          // 禁用自动更新
  "autoupdate": "notify"        // 仅通知有新版本，不自动更新
}
```

配置文件位置（按优先级）：

1. `~/.config/opencode/config.json`
2. `~/.config/opencode/opencode.json`
3. `~/.config/opencode/opencode.jsonc`
4. 项目根目录下的 `opencode.json` 或 `opencode.jsonc`

## 环境变量

可通过环境变量禁用自动更新：

```bash
export OPENCODE_DISABLE_AUTOUPDATE=1
opencode  # 不会自动更新
```

## 手动升级命令

用户也可以手动执行升级命令：

```bash
opencode upgrade                # 升级到最新版本
opencode upgrade 0.1.48         # 升级到指定版本
opencode upgrade -m bun         # 使用特定包管理器升级
```

## 版本信息

**文件位置**: `packages/opencode/src/installation/index.ts:163`

```typescript
export const VERSION = typeof OPENCODE_VERSION === "string" ? OPENCODE_VERSION : "local"
export const CHANNEL = typeof OPENCODE_CHANNEL === "string" ? OPENCODE_CHANNEL : "local"
```

版本和通道在编译时通过构建脚本注入，可通过以下方式查看：

```bash
opencode --version
```

## 事件通知

更新过程会发布事件，其他模块可以监听这些事件：

```typescript
// 新版本可用
Bus.publish(Installation.Event.UpdateAvailable, { version: latest })

// 更新完成
Bus.publish(Installation.Event.Updated, { version: latest })
```

## 关键文件清单

| 文件路径                                      | 功能                             |
| --------------------------------------------- | -------------------------------- |
| `packages/opencode/bin/opencode`              | CLI入口，查找并执行二进制文件    |
| `packages/opencode/src/cli/upgrade.ts`        | 版本检查和更新的主逻辑           |
| `packages/opencode/src/cli/cmd/tui/spawn.ts`  | TUI启动时触发版本检查            |
| `packages/opencode/src/cli/cmd/tui/worker.ts` | TUI worker中的版本检查           |
| `packages/opencode/src/installation/index.ts` | 安装方式检测、版本获取、更新执行 |
| `packages/opencode/src/config/config.ts`      | 配置管理，包括autoupdate设置     |

## 总结

OpenCode的自动更新机制设计如下：

1. **触发时机**：每次使用TUI启动时自动检查
2. **版本源**：根据安装方式从对应的源获取（NPM registry、Brew API、GitHub releases）
3. **安装方式检测**：自动识别是通过curl/npm/pnpm/bun/brew安装的
4. **配置控制**：支持自动更新、禁用更新、仅通知三种模式
5. **更新执行**：使用对应的包管理器执行更新命令
6. **错误处理**：失败时静默处理，不影响正常使用

这种设计确保用户始终使用最新版本，同时提供了足够的灵活性让用户控制更新行为。
