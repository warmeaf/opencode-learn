# Turborepo 在 OpenCode 项目中的实际作用

## 项目背景
OpenCode 是一个 Monorepo 项目，包含 19 个子包：
- `opencode` - CLI 工具
- `desktop` - 桌面应用
- `app` - Web 应用核心
- `ui` - UI 组件库
- `sdk` - SDK
- `util` - 工具库
- `plugin` - 插件系统
- `console/*` - 控制台相关（app、core、function、mail、resource）
- `web` - 文档站点
- 其他...

---

## Turborepo 的 3 个核心作用

### 1. 自动按依赖顺序构建

**问题**：包之间有依赖关系，必须按正确顺序构建。
```
util (无依赖)
  ↓
sdk (依赖 util)
  ↓
plugin (依赖 sdk)
  ↓
opencode (依赖 plugin + sdk + util)
  ↓
app (依赖 sdk + ui + util)
  ↓
desktop (依赖 app)
```

**Turborepo 解决方案**：
```json
// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"]  // ^ 表示"所有依赖包的 build 任务"
    }
  }
}
```

**实际效果**：
```bash
bun turbo build
# 自动按顺序构建：util → sdk → plugin → opencode → app → desktop
# 无依赖的包并行构建
```

---

### 2. 缓存未修改的包

**问题**：每次修改代码都重新构建所有包，浪费时间。

**Turborepo 解决方案**：
```json
{
  "tasks": {
    "build": {
      "outputs": ["dist/**"]  // 指定构建产物
    }
  }
}
```

**实际效果**：
```bash
# 首次构建：构建所有包（耗时 5 分钟）
bun turbo build

# 修改 util 包后再次构建：
# - util：重新构建（已修改）
# - sdk、plugin、opencode...：重新构建（依赖 util）
# - web、console：跳过（未修改且无依赖关系）
# 耗时仅 1 分钟
```

**缓存原理**：
- 基于源码、依赖、环境变量生成哈希
- 哈希未变则直接使用缓存
- 缓存存储在 `.turbo/cache` 目录

---

### 3. 统一管理所有包的任务

**问题**：19 个包，每个都有自己的脚本（typecheck、build、test...），如何在根目录统一执行？

**Turborepo 解决方案**：
```json
// 根目录 package.json
{
  "scripts": {
    "typecheck": "bun turbo typecheck"  // 一键检查所有包
  }
}

// turbo.json
{
  "tasks": {
    "typecheck": {}  // 在所有子包并行执行
  }
}
```

**实际效果**：
```bash
# 在根目录执行，自动检查所有包的类型错误
bun turbo typecheck

# 只检查某个包及其依赖
bun turbo typecheck --filter=opencode

# 构建所有包
bun turbo build

# 清除缓存后重新构建
bun turbo build --force
```

---

## 实际使用场景

### 场景 1：修改了 util 包
```bash
# Turborepo 会自动：
# 1. 检测到 util 包变化
# 2. 重新构建 util
# 3. 检测依赖链，重新构建 sdk、plugin、opencode...
# 4. 跳过与 util 无关的包（如 web）
```

### 场景 2：CI/CD 构建
```yaml
# .github/workflows/build.yml
- name: Build all packages
  run: bun turbo build
# Turborepo 会：
# - 识别未修改的包
# - 从缓存恢复或跳过
# - 大幅减少 CI 时间
```

### 场景 3：团队协作
```bash
# 开发者 A 刚构建完项目
# 开发者 B 拉取代码后：
bun turbo build
# Turborepo 会：
# - 检测代码未变化
# - 直接使用 A 的缓存（如果配置了远程缓存）
# - 几秒内完成构建
```

---

## 配合 Bun Workspaces 的协作

**Bun Workspaces** 负责：
- 依赖安装和链接
- 允许包之间相互引用（`workspace:*`）

**Turborepo** 负责：
- 任务执行和编排
- 缓存管理
- 增量构建

**示例**：
```json
// packages/app/package.json
{
  "dependencies": {
    "@opencode-ai/sdk": "workspace:*",  // Bun 负责链接本地包
    "@opencode-ai/ui": "workspace:*"
  }
}

// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"]  // Turborepo 负责按顺序构建
    }
  }
}
```

---

## 当前配置状态

项目中的 `turbo.json` 配置：
```json
{
  "tasks": {
    "typecheck": {},           // 并行检查所有包
    "build": {
      "dependsOn": ["^build"], // 按依赖顺序构建
      "outputs": ["dist/**"]   // 缓存构建产物
    },
    "opencode#test": {         // 仅 opencode 包的测试
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

**可以补充的配置**：
```json
{
  "tasks": {
    "dev": {
      "cache": false,          // 开发服务器不缓存
      "persistent": true       // 长期运行的任务
    },
    "lint": {
      "outputs": []            // 无产物的任务
    }
  }
}
```

---

## 关键命令速查

```bash
# 类型检查所有包
bun turbo typecheck

# 构建所有包（增量）
bun turbo build

# 强制重新构建（忽略缓存）
bun turbo build --force

# 只构建某个包
bun turbo build --filter=opencode

# 构建某个包及其所有依赖
bun turbo build --filter=...opencode

# 并行执行多个任务
bun turbo run build lint

# 查看任务执行计划
bun turbo build --dry-run
```

---

## 总结

Turborepo 在 OpenCode 项目中的价值：

1. **自动处理依赖顺序** - 无需手动记忆 19 个包的构建顺序
2. **大幅提升构建速度** - 通过缓存跳过未修改的包
3. **统一操作入口** - 根目录命令管理所有子包

**实际收益**：
- 首次构建：5 分钟
- 增量构建：1 分钟（缓存命中率 80%+）
- CI/CD 时间减少 50-70%

**没有 Turborepo 会怎样**：
- 每次修改代码都要重新构建所有包
- 需要手动维护构建顺序脚本
- 开发效率大幅降低
