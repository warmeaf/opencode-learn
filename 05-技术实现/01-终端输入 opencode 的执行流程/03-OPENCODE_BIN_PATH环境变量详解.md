# OPENCODE_BIN_PATH 环境变量详解

## 0. 前置说明

**工作目录**: 以下所有命令假设在 `packages/opencode` 目录下执行。如果在项目根目录执行，请在路径前添加 `packages/opencode/`。

---

## 1. 变量来源

OPENCODE_BIN_PATH 是一个**开发模式**环境变量，由开发者手动设置，用于指定 OpenCode CLI 二进制文件的自定义路径。

### 设置方式

#### Unix/macOS

```bash
# 临时设置（单次执行）
OPENCODE_BIN_PATH=./dist/opencode-darwin-arm64/bin/opencode opencode

# 设置环境变量（当前会话）
export OPENCODE_BIN_PATH=./dist/opencode-darwin-arm64/bin/opencode

# 使用 direnv
echo 'export OPENCODE_BIN_PATH=./dist/opencode-darwin-arm64/bin/opencode' > .envrc
direnv allow
```

#### Windows

```powershell
# PowerShell
$env:OPENCODE_BIN_PATH=".\dist\opencode-windows-x64\bin\opencode.exe"
opencode

# 临时设置（单次执行）
$env:OPENCODE_BIN_PATH=".\dist\opencode-windows-x64\bin\opencode.exe"; opencode
```

```cmd
# CMD
set OPENCODE_BIN_PATH=.\dist\opencode-windows-x64\bin\opencode.exe
opencode
```

> **注意**: Windows 平台上的二进制文件带有 `.exe` 扩展名

---

## 2. 代码位置与实现

### 核心代码位置

**文件**: `packages/opencode/bin/opencode` (第20-23行)

```javascript
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath) // 直接执行指定的二进制
}
```

### 完整上下文

```javascript
#!/usr/bin/env node

const childProcess = require("child_process")
const fs = require("fs")
const path = require("path")
const os = require("os")

function run(target) {
  const result = childProcess.spawnSync(target, process.argv.slice(2), {
    stdio: "inherit",
  })
  if (result.error) {
    console.error(result.error.message)
    process.exit(1)
  }
  const code = typeof result.status === "number" ? result.status : 0
  process.exit(code)
}

// 【关键】优先检查环境变量
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath) // 注意：这里会直接退出，不会继续执行后续代码
}

// 如果没有设置环境变量，则执行平台检测和查找
const scriptPath = fs.realpathSync(__filename)
const scriptDir = path.dirname(scriptPath)

// ... 平台检测和查找逻辑 ...
```

---

## 3. 作用机制

### 执行流程

```
用户执行: opencode
        │
        ▼
┌───────────────────────────────┐
│  bin/opencode (Shell Wrapper)  │
└───────────────┬───────────────┘
                │
                ▼
        检查 OPENCODE_BIN_PATH ?
        │
    ┌───┴───┐
    │ 否    │ 是  ← 开发模式走这个分支
    ▼       ▼
 平台检测   直接执行指定二进制
    │       (跳过所有检测和查找)
    ▼
 自动查找二进制
    │
    ▼
 执行查找到的二进制
```

### 关键特性

1. **优先级最高**: 在所有平台检测和查找逻辑之前执行
2. **完全绕过**: 设置后会跳过自动检测、平台匹配、查找算法
3. **直接执行**: 直接调用 `run()` 函数执行指定路径的二进制
4. **即时生效**: 一旦设置，立即生效，无需重新安装

---

## 4. 使用场景

### 场景 1: 本地开发

在开发 OpenCode 本身时，测试本地编译的二进制：

**Unix/macOS**:

```bash
# 进入项目目录
cd packages/opencode

# 编译当前平台的二进制
bun run build --single

# 使用编译的二进制
export OPENCODE_BIN_PATH=./dist/opencode-$(uname -s | tr '[:upper:]' '[:lower:]')-$(uname -m)/bin/opencode
opencode --version
```

**Windows**:

```powershell
# 进入项目目录
cd packages\opencode

# 编译当前平台的二进制
bun run build --single

# 使用编译的二进制
$env:OPENCODE_BIN_PATH=".\dist\opencode-windows-x64\bin\opencode.exe"
opencode --version
```

### 场景 2: 调试特定版本

调试特定平台或变体的二进制：

```bash
# 测试 baseline 版本（无 AVX2）
export OPENCODE_BIN_PATH=./dist/opencode-linux-x64-baseline/bin/opencode
opencode

# 测试 musl 版本（Alpine 兼容）
export OPENCODE_BIN_PATH=./dist/opencode-linux-x64-musl/bin/opencode
opencode
```

### 场景 3: 跨平台测试

在当前平台上测试其他平台的二进制（需要使用 Docker 或模拟器）：

```bash
# 在容器中测试 Linux 二进制
docker run --rm \
  -v $(pwd)/dist/opencode-linux-x64/bin/opencode:/usr/local/bin/opencode \
  ubuntu:latest \
  opencode --version

# 或者在容器中使用 OPENCODE_BIN_PATH
docker run --rm \
  -v $(pwd)/dist/opencode-linux-x64/bin/opencode:/opencode \
  -e OPENCODE_BIN_PATH=/opencode \
  ubuntu:latest \
  opencode --version
```

### 场景 4: CI/CD 测试

在 CI/CD 流程中测试不同的构建产物：

```yaml
# .github/workflows/test.yml
- name: Test arm64 binary
  run: |
    export OPENCODE_BIN_PATH=./dist/opencode-darwin-arm64/bin/opencode
    bun run test
```

---

## 5. 与生产模式的区别

### 生产模式（默认行为）

```javascript
// 1. 检测平台
const platform = platformMap[os.platform()] || os.platform()
const arch = archMap[os.arch()] || os.arch()

// 2. 构建包名
const base = `opencode-${platform}-${arch}`

// 3. 向上遍历 node_modules 查找
const resolved = findBinary(scriptDir)

// 4. 执行
run(resolved)
```

**特点**:

- 自动检测平台和架构
- 模糊匹配支持变体（baseline、musl）
- 向上遍历 node_modules 树查找
- 友好的错误提示

### 开发模式（使用 OPENCODE_BIN_PATH）

```javascript
// 直接使用指定的路径
const envPath = process.env.OPENCODE_BIN_PATH
if (envPath) {
  run(envPath)
}
```

**特点**:

- 无需安装，直接使用本地编译的二进制
- 跳过所有检测和查找逻辑
- 适合快速迭代和调试

---

## 6. 注意事项

### ⚠️ 重要提示

1. **不要在生产环境使用**: 这个变量主要用于开发，生产环境应使用 npm 安装的标准二进制

2. **路径必须存在**: 如果路径不存在，会报错并退出：

   ```javascript
   if (result.error) {
     console.error(result.error.message)
     process.exit(1)
   }
   ```

3. **二进制必须可执行**: 在 Unix/macOS 上确保有执行权限（Bun 构建通常会自动设置）：

   ```bash
   chmod +x ./dist/opencode-darwin-arm64/bin/opencode
   ```

4. **Windows 平台注意事项**:
   - 二进制文件名为 `opencode.exe`
   - 路径分隔符使用反斜杠 `\` 或正斜杠 `/` 都可以
   - 在 PowerShell 中使用 `$env:VARIABLE` 语法
   - 在 CMD 中使用 `set VARIABLE` 语法

5. **参数传递**: 所有命令行参数都会传递给指定的二进制：

   ```javascript
   run(envPath) // 等同于 run(process.env.OPENCODE_BIN_PATH)
   ```

6. **退出码传递**: 二进制的退出码会正确传递给父进程

---

## 7. 实际示例

### 示例 1: 开发工作流

```bash
# 1. 克隆仓库
git clone https://github.com/sst/opencode.git
cd opencode/packages/opencode

# 2. 安装依赖
bun install

# 3. 编译当前平台的二进制
bun run build --single

# 4. 设置环境变量（Unix/macOS）
export OPENCODE_BIN_PATH=./dist/opencode-$(uname -s | tr '[:upper:]' '[:lower:]')-$(uname -m)/bin/opencode

# 4. 设置环境变量（Windows）
$env:OPENCODE_BIN_PATH=".\dist\opencode-windows-x64\bin\opencode.exe"

# 5. 测试
opencode --version
opencode run "hello world"
```

### 示例 2: 测试不同变体

```bash
# 编译所有平台
bun run build

# 测试标准版
export OPENCODE_BIN_PATH=./dist/opencode-linux-x64/bin/opencode
opencode --version

# 测试 baseline 版本（无 AVX2，兼容旧 CPU）
export OPENCODE_BIN_PATH=./dist/opencode-linux-x64-baseline/bin/opencode
opencode --version

# 测试 musl 版本（Alpine Linux 兼容）
export OPENCODE_BIN_PATH=./dist/opencode-linux-x64-musl/bin/opencode
opencode --version
```

### 示例 3: Docker 多平台测试

```bash
# 编译所有平台
bun run build

# 在 Alpine Linux 容器中测试 musl 版本
docker run --rm \
  -v $(pwd)/dist/opencode-linux-x64-musl/bin/opencode:/usr/local/bin/opencode \
  alpine:latest \
  opencode --version
```

---

## 8. 相关代码位置

### 核心文件

1. **Shell Wrapper**: `packages/opencode/bin/opencode:20-23`
   - 环境变量检查
   - 平台检测
   - 二进制查找
   - 执行逻辑

2. **文档**: `docs-learn/05-技术实现/01-终端输入 opencode 的执行流程/02-二进制文件查找与执行机制.md`
   - 详细的技术文档
   - 使用示例
   - 常见问题解答

### 相关环境变量

| 环境变量            | 用途                       |
| ------------------- | -------------------------- |
| `OPENCODE_BIN_PATH` | 指定二进制路径（开发模式） |
| `AGENT`             | 标识运行在 agent 模式      |
| `OPENCODE`          | 标识是 OpenCode 环境       |

---

## 9. 总结

OPENCODE_BIN_PATH 是一个专为**开发场景**设计的环境变量，允许开发者：

✅ **快速测试** 本地编译的二进制，无需发布和安装
✅ **灵活调试** 不同平台和变体的二进制
✅ **简化开发** 工作流，提高开发效率
✅ **跳过检测** 自动平台检测和查找逻辑

**核心原则**:

- 仅在开发时使用
- 不要在生产环境设置
- 确保路径正确且可执行
- Windows 平台记得使用 `.exe` 扩展名
- 所有参数和退出码正确传递

---

## 参考资源

- [二进制文件查找与执行机制](./02-二进制文件查找与执行机制.md)
- [终端输入 opencode 的执行流程](./01-终端输入 opencode 的执行流程.md)
- [Shell Wrapper 源码](../../../packages/opencode/bin/opencode)
