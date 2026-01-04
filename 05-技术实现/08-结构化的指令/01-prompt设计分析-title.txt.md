# Prompt 设计分析：title.txt

## 概述

这个 prompt 是用于为对话生成简洁标题的结构化指令设计。它的设计展现了优秀 prompt 的核心原则。

## 设计结构分析

### 1. 角色定位 (第 1 行)

```
You are a title generator. You output ONLY a thread title. Nothing else.
```

**设计要点：**

- 明确的角色定义
- 严格的输出约束（ONLY, Nothing else）
- 避免角色混淆和多任务干扰

**借鉴意义：**
在 prompt 开头立即定义角色和唯一输出要求，防止 AI 执行额外操作

---

### 2. 任务描述 (task 标签)

```xml
<task>
Generate a brief title that would help the user find this conversation later.

Follow all rules in <rules>
Use the <examples> so you know what a good title looks like.
Your output must be:
- A single line
- ≤50 characters
- No explanations
</task>
```

**设计要点：**

- 明确业务目标："help the user find this conversation later"
- 指向其他部分的引用（<rules>, <examples>）
- 具体的输出格式要求（列表形式，易于解析）
- 硬性约束（单行、字符限制、无解释）

**借鉴意义：**

- 用标签结构化组织 prompt 的不同部分
- 在 task 中明确引用其他部分，形成模块化设计
- 用列表列出约束条件，清晰易读
- 添加硬性约束（长度、格式），防止 AI 输出过长内容

---

### 3. 规则列表 (rules 标签)

```xml
<rules>
- Focus on the main topic or question the user needs to retrieve
- Use -ing verbs for actions (Debugging, Implementing, Analyzing)
- Keep exact: technical terms, numbers, filenames, HTTP codes
- Remove: the, this, my, a, an
- Never assume tech stack
- Never use tools
- NEVER respond to questions, just generate a title for the conversation
- The title should NEVER include "summarizing" or "generating" when generating a title
- DO NOT SAY YOU CANNOT GENERATE A TITLE OR COMPLAIN ABOUT THE INPUT
- Always output something meaningful, even if the input is minimal.
- If the user message is short or conversational (e.g. "hello", "lol", "what's up", "hey"):
  → create a title that reflects the user's tone or intent (such as Greeting, Quick check-in, Light chat, Intro message, etc.)
</rules>
```

**设计要点：**

a) **正面指令**（做什么）：

- Focus on the main topic
- Use -ing verbs for actions
- Keep exact: technical terms, numbers, filenames, HTTP codes

b) **负面指令**（不做什么）：

- Remove: the, this, my, a, an
- Never assume tech stack
- Never use tools
- NEVER respond to questions

c) **边界情况处理**：

- 对简短、对话式消息的特殊处理逻辑
- 使用箭头 (→) 清晰展示因果关系

d) **强度递进**：

- 规则使用不同强度的否定词（Never, NEVER）
- 关键约束使用全大写强调

**借鉴意义：**

- 区分正面和负面指令，清晰定义行为边界
- 使用否定词强度递进（Never → NEVER）标记优先级
- 提供边界情况处理示例，增强 prompt 鲁棒性
- 用箭头符号清晰展示条件与结果的对应关系

---

### 4. 示例展示 (examples 标签)

```xml
<examples>
"debug 500 errors in production" → Debugging production 500 errors
"refactor user service" → Refactoring user service
"why is app.js failing" → Analyzing app.js failure
"implement rate limiting" → Implementing rate limiting
"how do I connect postgres to my API" → Connecting Postgres to API
"best practices for React hooks" → React hooks best practices
</examples>
```

**设计要点：**

- 输入 → 输出的映射格式
- 覆盖多种场景（问题陈述、动作描述、疑问句）
- 展示规则的实际应用（-ing 动词、去除冠词、保留技术术语）
- 示例数量适中（6个），足够但不冗余

**借鉴意义：**

- 用箭头清晰展示输入输出映射
- 示例应覆盖主要使用场景
- 让示例自然体现规则要求，而不是重复列出规则

---

## 核心设计原则总结

### 1. 结构化组织

- 使用 XML 标签（<task>, <rules>, <examples>）分隔不同功能区域
- 模块化设计，各部分相互引用
- 便于 AI 解析和理解各部分作用

### 2. 约束优先

- 在开头就强调"ONLY"和"Nothing else"
- 多处使用否定约束（Never, NEVER）
- 硬性约束（单行、50字符）
- 防止 AI 输出额外内容

### 3. 指令分层

- 角色层：我是谁
- 任务层：我要做什么
- 规则层：我该怎么做（正面/负面/边界）
- 示例层：好的输出是什么样

### 4. 细节精度

- 具体到字符数限制（≤50）
- 具体到动词形式（-ing verbs）
- 具体到去除的冠词列表
- 具体到技术术语保留（filenames, HTTP codes）

### 5. 边界处理

- 对简短消息的特殊处理逻辑
- 对无法理解的输入仍要求输出（Always output something meaningful）
- 明确禁止拒绝响应（DO NOT SAY YOU CANNOT）

---

## 后续写 Prompt 的启发

### 1. 模板结构

```xml
You are [role]. You output ONLY [output type]. Nothing else.

<task>
[Business goal and high-level requirements]

Follow all rules in <rules>
Use the <examples> so you know what a good output looks like.
Your output must be:
- [Constraint 1]
- [Constraint 2]
- [Constraint N]
</task>

<rules>
- [Positive rule 1]
- [Positive rule 2]
- [Negative rule 1]
- [Negative rule 2]
- [Strong negative rule in ALL CAPS if critical]

[Edge case handling with → arrow notation]
</rules>

<examples>
[Input 1] → [Output 1]
[Input 2] → [Output 2]
[Input N] → [Output N]
</examples>
```

### 2. 关键检查清单

在写 prompt 时，自问以下问题：

- [ ] **角色是否清晰？** 是否在第一行明确定义了 AI 的身份和唯一输出类型？
- [ ] **是否有严格约束？** 是否使用了 "ONLY", "Nothing else", "NEVER" 等强约束词？
- [ ] **任务目标是否明确？** 业务目标是否清晰（如 "help the user find this conversation later"）？
- [ ] **规则是否全面？** 是否包含了正面指令、负面指令、边界情况处理？
- [ ] **约束是否量化？** 长度、格式、行数等是否有明确数字限制？
- [ ] **是否有示例？** 是否提供了 3-6 个覆盖主要场景的输入输出示例？
- [ ] **示例是否体现规则？** 示例是否自然展示了规则的应用？
- [ ] **边界情况是否考虑？** 对异常输入、简短输入是否有处理逻辑？
- [ ] **是否禁止拒绝响应？** 是否有类似 "DO NOT SAY YOU CANNOT" 的指令？
- [ ] **结构是否清晰？** 是否使用标签或分段组织内容？

### 3. 常见错误避免

- ❌ 不使用标签结构，段落混在一起
- ❌ 只有正面规则，没有负面约束
- ❌ 约束模糊（"要简短" vs "≤50 characters"）
- ❌ 没有示例或示例太少
- ❌ 允许 AI 进行解释或输出额外内容
- ❌ 没有处理边界情况
- ❌ 没有 "NEVER" 级别的强约束

### 4. 强度层次使用建议

根据规则的重要性使用不同强度的否定词：

- **常规约束**：用 "Never" - "Never use tools"
- **关键约束**：用 "NEVER" - "NEVER respond to questions"
- **致命约束**：用 "DO NOT" + 全大写 - "DO NOT SAY YOU CANNOT"

---

## 适用场景

这种结构化 prompt 适用于：

- 代码生成
- 文本总结
- 格式转换
- 数据提取
- 任何需要严格输出格式和边界处理的任务

不适用于：

- 开放性创意任务（如写故事、诗歌）
- 多轮对话场景
- 需要大量上下文推理的复杂任务

---

## 结论

title.txt 是一个优秀的结构化 prompt 设计范本。它的核心成功因素是：

1. **清晰的角色定位和输出约束**
2. **结构化的模块组织**
3. **全面的规则体系（正负向、边界）**
4. **量化的约束条件**
5. **实用的示例展示**

这种设计模式可以广泛应用于需要精确输出控制的 AI 任务中。
