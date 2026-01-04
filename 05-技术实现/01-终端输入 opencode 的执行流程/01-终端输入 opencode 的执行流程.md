# 终端输入 opencode 的执行流程

## 目录
- [1. 整体架构](#1-整体架构)
- [2. 从命令行到TUI的完整流程](#2-从命令行到tui的完整流程)
- [3. TUI 组件系统](#3-tui-组件系统)

---

## 1. 整体架构

OpenCode CLI 采用 **Worker + RPC** 的分层架构，将 UI 渲染和业务逻辑分离：

```
┌─────────────────────────────────────────────────────────────┐
│                    用户输入: `opencode`                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Shell Wrapper Layer (bin/opencode)                      │
│     - 支持 OPENCODE_BIN_PATH 环境变量                         │
│     - 查找并执行对应二进制文件                                  │
│     - 检测平台和架构                                          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  2. CLI Framework Layer (src/index.ts)                       │
│     - Yargs 命令行解析                                       │
│     - 命令注册和路由                                         │
│     - 中间件处理（日志、环境变量）                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  3. TUI Command Layer (src/cli/cmd/tui/thread.ts)            │
│     - 解析命令参数                                            │
│     - 切换到项目目录                                          │
│     - 创建 Web Worker                                        │
│     - 建立 RPC 通信                                           │
└────────────────────────┬────────────────────────────────────┘
                         │
            ┌────────────┴────────────┐
            │                         │
            ▼                         ▼
┌──────────────────────┐   ┌──────────────────────────────┐
│  Main Thread (UI)    │   │  Worker Thread (Backend)      │
│  - SolidJS 组件      │◄─►│  - HTTP/WebSocket Server      │
│  - OpenTUI 渲染      │ RPC│  - AI SDK 集成               │
│  - 用户交互          │   │  - 文件系统操作               │
│  - 事件响应          │   │  - 工具执行                   │
└──────────────────────┘   └──────────────────────────────┘
```

---

## 2. 从命令行到TUI的完整流程

### 2.1 入口：Shell Wrapper

**文件**: `packages/opencode/bin/opencode`

当你全局安装 opencode-ai 后，npm 会在系统 PATH 中创建 `opencode` 命令，指向这个文件。

**核心逻辑**:

```javascript
// 1. 检查环境变量
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath)  // 开发模式：直接执行指定的二进制
}

// 2. 自动检测平台和架构
const platformMap = {
  darwin: "darwin",
  linux: "linux",
  win32: "windows",
}
const archMap = {
  x64: "x64",
  arm64: "arm64",
  arm: "arm",
}

const binaryName = `opencode-${platform}-${arch}`
const binaryFile = platform === "windows" ? "opencode.exe" : "opencode"

// 3. 查找二进制文件
// 从当前目录向上遍历 node_modules，查找匹配的平台包
function findBinary(startDir) {
  let current = startDir
  for (;;) {
    const modules = path.join(current, "node_modules")
    // 查找 opencode-darwin-x64 这样的包
    // ...
  }
}

// 4. 执行二进制
run(resolvedBinaryPath)
```

**关键点**:
- 使用 `spawnSync` 以 `stdio: "inherit"` 模式执行，让子进程继承 stdin/stdout/stderr
- 支持跨平台：Windows 用 `.exe`，Unix 用可执行文件

---

### 2.2 CLI 命令解析

**文件**: `packages/opencode/src/index.ts`

使用 **Yargs** 框架构建命令行界面。

**命令注册**:

```typescript
const cli = yargs(hideBin(process.argv))
  .scriptName("opencode")
  .wrap(100)
  .command(TuiThreadCommand)      // 默认命令: $0 [project]
  .command(RunCommand)            // 单次执行: run [message]
  .command(GenerateCommand)       // 代码生成: generate
  .command(AuthCommand)           // 认证: auth
  // ... 更多命令
```

**中间件**:

```typescript
.middleware(async (opts) => {
  // 初始化日志系统
  await Log.init({
    print: process.argv.includes("--print-logs"),
    dev: Installation.isLocal(),
    level: opts.logLevel || (Installation.isLocal() ? "DEBUG" : "INFO"),
  })

  // 设置环境变量，供工具识别
  process.env.AGENT = "1"
  process.env.OPENCODE = "1"

  Log.Default.info("opencode", {
    version: Installation.VERSION,
    args: process.argv.slice(2),
  })
})
```

**错误处理**:

```typescript
.fail((msg) => {
  // 未知参数或参数不足时显示帮助
  if (msg.startsWith("Unknown argument")) {
    cli.showHelp("log")
  }
  process.exit(1)
})
```

---

### 2.3 TUI 命令启动

**文件**: `packages/opencode/src/cli/cmd/tui/thread.ts`

这是默认命令，当你输入 `opencode` 时执行。

**参数解析**:

```typescript
export const TuiThreadCommand = cmd({
  command: "$0 [project]",        // $0 表示默认命令
  describe: "start opencode tui",
  builder: (yargs) =>
    withNetworkOptions(yargs)
      .positional("project", {
        type: "string",
        describe: "path to start opencode in",
      })
      .option("model", {
        type: "string",
        alias: ["m"],
        describe: "model to use in the format of provider/model",
      })
      .option("continue", {
        alias: ["c"],
        describe: "continue the last session",
        type: "boolean",
      })
      .option("session", {
        alias: ["s"],
        type: "string",
        describe: "session id to continue",
      })
      .option("prompt", {
        type: "string",
        describe: "prompt to use",
      })
      .option("agent", {
        type: "string",
        describe: "agent to use",
      }),
```

**Handler 实现**:

```typescript
handler: async (args) => {
  // 1. 解析工作目录
  const baseCwd = process.env.PWD ?? process.cwd()
  const cwd = args.project
    ? path.resolve(baseCwd, args.project)
    : process.cwd()

  // 2. 切换到项目目录
  try {
    process.chdir(cwd)
  } catch (e) {
    UI.error("Failed to change directory to " + cwd)
    return
  }

  // 3. 创建 Worker
  const worker = new Worker(workerPath, {
    env: Object.fromEntries(
      Object.entries(process.env).filter(
        (entry): entry is [string, string] => entry[1] !== undefined
      )
    ),
  })

  // 4. 建立 RPC 通信
  const client = Rpc.client<typeof rpc>(worker)

  // 5. 启动服务器
  const opts = await resolveNetworkOptions(args)
  const server = await client.call("server", opts)

  // 6. 处理输入（管道或参数）
  const prompt = await iife(async () => {
    const piped = !process.stdin.isTTY ? await Bun.stdin.text() : undefined
    if (!args.prompt) return piped
    return piped ? piped + "\n" + args.prompt : args.prompt
  })

  // 7. 启动 TUI
  await tui({
    url: server.url,           // WebSocket 服务器地址
    args: {
      continue: args.continue,
      sessionID: args.session,
      agent: args.agent,
      model: args.model,
      prompt,
    },
    onExit: async () => {
      await client.call("shutdown", undefined)
    },
  })
}
```

---

### 2.4 Worker 后端服务

**文件**: `packages/opencode/src/cli/cmd/tui/worker.ts`

Worker 运行在独立线程，负责后端逻辑。

**RPC 接口定义**:

```typescript
export const rpc = {
  // 启动 HTTP/WebSocket 服务器
  async server(input: { port: number; hostname: string; mdns?: boolean }) {
    if (server) await server.stop(true)
    server = Server.listen(input)
    return {
      url: server.url.toString(),
    }
  },

  // 检查更新
  async checkUpgrade(input: { directory: string }) {
    await Instance.provide({
      directory: input.directory,
      init: InstanceBootstrap,
      fn: async () => {
        await upgrade().catch(() => {})
      },
    })
  },

  // 关闭服务
  async shutdown() {
    Log.Default.info("worker shutting down")
    await Instance.disposeAll()
    server.stop(true)
  },
}

// 监听来自主线程的 RPC 请求
Rpc.listen(rpc)
```

**RPC 通信机制**:

**文件**: `packages/opencode/src/util/rpc.ts`

```typescript
export namespace Rpc {
  // Worker 端：监听并处理请求
  export function listen(rpc: Definition) {
    onmessage = async (evt) => {
      const parsed = JSON.parse(evt.data)
      if (parsed.type === "rpc.request") {
        const result = await rpc[parsed.method](parsed.input)
        postMessage(JSON.stringify({
          type: "rpc.result",
          result,
          id: parsed.id
        }))
      }
    }
  }

  // 主线程端：发送请求并接收响应
  export function client<T extends Definition>(target: Worker) {
    const pending = new Map<number, (result: any) => void>()
    let id = 0

    target.onmessage = async (evt) => {
      const parsed = JSON.parse(evt.data)
      if (parsed.type === "rpc.result") {
        const resolve = pending.get(parsed.id)
        if (resolve) {
          resolve(parsed.result)
          pending.delete(parsed.id)
        }
      }
    }

    return {
      call<Method extends keyof T>(
        method: Method,
        input: Parameters<T[Method]>[0]
      ): Promise<ReturnType<T[Method]>> {
        const requestId = id++
        return new Promise((resolve) => {
          pending.set(requestId, resolve)
          target.postMessage(JSON.stringify({
            type: "rpc.request",
            method,
            input,
            id: requestId
          }))
        })
      },
    }
  }
}
```

---

### 2.5 TUI 渲染

**文件**: `packages/opencode/src/cli/cmd/tui/app.tsx`

使用 **OpenTUI + SolidJS** 构建终端用户界面。

**主渲染函数**:

```typescript
export function tui(input: { url: string; args: Args; onExit?: () => Promise<void> }) {
  return new Promise<void>(async (resolve) => {
    // 1. 检测终端背景色（暗色/亮色模式）
    const mode = await getTerminalBackgroundColor()

    // 2. 定义退出回调
    const onExit = async () => {
      await input.onExit?.()
      resolve()
    }

    // 3. 渲染 SolidJS 应用
    render(
      () => (
        <ErrorBoundary fallback={(error, reset) => (
          <ErrorComponent error={error} reset={reset} onExit={onExit} mode={mode} />
        )}>
          <ArgsProvider {...input.args}>
            <ExitProvider onExit={onExit}>
              <KVProvider>
                <ToastProvider>
                  <RouteProvider>
                    <SDKProvider url={input.url}>
                      <SyncProvider>
                        <ThemeProvider mode={mode}>
                          <LocalProvider>
                            <KeybindProvider>
                              <PromptStashProvider>
                                <DialogProvider>
                                  <CommandProvider>
                                    <PromptHistoryProvider>
                                      <PromptRefProvider>
                                        <App />
                                      </PromptRefProvider>
                                    </PromptHistoryProvider>
                                  </CommandProvider>
                                </DialogProvider>
                              </PromptStashProvider>
                            </KeybindProvider>
                          </LocalProvider>
                        </ThemeProvider>
                      </SyncProvider>
                    </SDKProvider>
                  </RouteProvider>
                </ToastProvider>
              </KVProvider>
            </ExitProvider>
          </ArgsProvider>
        </ErrorBoundary>
      ),
      {
        targetFps: 60,
        gatherStats: false,
        exitOnCtrlC: false,
        useKittyKeyboard: {},        // 支持 Kitty 键盘协议
        consoleOptions: {
          keyBindings: [{
            name: "y",
            ctrl: true,
            action: "copy-selection"
          }],
          onCopySelection: (text) => {
            Clipboard.copy(text).catch((error) => {
              console.error(`Failed to copy: ${error}`)
            })
          },
        },
      }
    )
  })
}
```

**Provider 层级说明**:

| Provider | 用途 |
|----------|------|
| `ArgsProvider` | 命令行参数（model, agent, sessionID 等） |
| `ExitProvider` | 退出处理 |
| `KVProvider` | 持久化键值存储 |
| `ToastProvider` | 通知提示 |
| `RouteProvider` | 路由管理（home/session） |
| `SDKProvider` | WebSocket SDK 客户端 |
| `SyncProvider` | 服务端数据同步（session, provider, mcp） |
| `ThemeProvider` | 主题（颜色、暗色/亮色模式） |
| `LocalProvider` | 本地状态（model, agent 选择） |
| `KeybindProvider` | 键盘快捷键 |
| `PromptStashProvider` | Prompt 草稿 |
| `DialogProvider` | 对话框管理 |
| `CommandProvider` | 命令面板（Ctrl+X） |
| `PromptHistoryProvider` | 历史记录 |
| `PromptRefProvider` | Prompt 引用 |

---

## 3. TUI 组件系统

**位置**: `packages/opencode/src/cli/cmd/tui/`

```
tui/
├── app.tsx                    # TUI 主入口
├── worker.ts                  # Web Worker（后端服务）
├── thread.ts                  # TUI 命令处理器
├── attach.ts                  # 附加到运行中的服务器
├── spawn.ts                   # 生成新进程
├── event.ts                   # 事件定义
├── component/                 # UI 组件
│   ├── logo.tsx              # Logo 组件
│   ├── prompt/               # 提示输入组件
│   │   ├── index.tsx         # 主组件
│   │   ├── history.tsx       # 历史记录
│   │   ├── stash.tsx         # 草稿
│   │   └── autocomplete.tsx  # 自动完成
│   ├── dialog-*.tsx          # 各种对话框
│   └── todo-item.tsx         # Todo 列表项
├── ui/                        # 通用 UI 组件
│   ├── dialog.tsx            # 对话框基础
│   ├── toast.tsx             # 通知提示
│   └── spinner.ts            # 加载动画
├── context/                   # React/Solid 上下文
│   ├── sdk.tsx               # SDK 客户端上下文
│   ├── sync.tsx              # 同步上下文
│   ├── local.tsx             # 本地状态
│   ├── theme.tsx             # 主题上下文
│   ├── keybind.tsx           # 键盘绑定
│   ├── route.tsx             # 路由上下文
│   ├── kv.tsx                # 键值存储
│   ├── exit.tsx              # 退出处理
│   ├── prompt.tsx            # Prompt 引用
│   ├── args.tsx              # 命令参数
│   └── directory.tsx         # 目录上下文
├── routes/                    # 路由页面
│   ├── home.tsx              # 主页（会话列表）
│   └── session/              # 会话页面
│       ├── index.tsx         # 主界面
│       ├── header.tsx        # 顶部栏
│       ├── footer.tsx        # 底部栏
│       └── sidebar.tsx       # 侧边栏
└── util/                      # 工具函数
    ├── clipboard.ts          # 剪贴板操作
    ├── editor.ts             # 编辑器集成
    └── terminal.ts           # 终端工具
```

---

## 核心流程总结

1. **Shell Wrapper** (`bin/opencode`) → 检测平台并执行对应二进制
2. **CLI 解析** (`src/index.ts`) → Yargs 路由到默认 TUI 命令
3. **TUI 命令** (`src/cli/cmd/tui/thread.ts`) → 创建 Worker、建立 RPC、启动服务器
4. **Worker 后端** (`src/cli/cmd/tui/worker.ts`) → 启动 HTTP/WebSocket 服务器
5. **TUI 渲染** (`src/cli/cmd/tui/app.tsx`) → OpenTUI + SolidJS 渲染交互式界面
