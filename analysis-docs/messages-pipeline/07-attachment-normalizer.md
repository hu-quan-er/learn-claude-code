# 07 attachment normalizer

> `normalizeAttachmentForAPI` 是一个 50+ case 的超长 switch，把每一种 attachment 类型转换成 user message。它本质上是一份**"系统能向模型说哪些话"的完整目录**。本篇做系统盘点。

## 7.1 attachment 是什么

回顾 [01](./01-消息类型与构造.md)：`attachment` 是 7 种 Message type 之一。它代表"系统主动注入对话流的内容"——不是用户发的、不是模型生成的。

attachment 在 store 里以 `AttachmentMessage` 形式存在，结构是：

```ts
{
  type: 'attachment',
  attachment: { type: '<具体类型>', ...类型相关字段 },
  uuid, timestamp, ...
}
```

normalize 主循环的 `case 'attachment'` 调 `normalizeAttachmentForAPI(message.attachment)` 把它**展开成 `UserMessage[]`**——之后 `ensureSystemReminderWrap` + 合并进前一条 user。

注意返回的是**数组**——一个 attachment 可能展开成 0 条（被忽略）、1 条、或多条 user message。

## 7.2 全部 attachment 类型清单

`normalizeAttachmentForAPI` 的 switch（`messages.ts:3524` 起）+ switch 前的 feature-gate 块，覆盖大约 50 种类型。按用途分 7 大类：

### 类 1：文件内容注入

| type | 作用 | 展开形式 |
|---|---|---|
| `file` | `@file` 提及的文件内容 | 伪造 FileRead tool_use + tool_result（4 子类：image/text/notebook/pdf） |
| `edited_text_file` | 文件被用户/linter 改过 | 单 user message（带 diff snippet） |
| `directory` | 目录列表 | 伪造 Bash `ls` tool_use + tool_result |
| `already_read_file` | 已读过的文件 | 引用提示 |
| `compact_file_reference` | 压缩前读过、内容太大没保留 | 提示"用 Read 重新读" |
| `pdf_reference` | 太大的 PDF | 提示"用 pages 参数分页读" |
| `nested_memory` | 嵌套 CLAUDE.md | 文件内容 user message |

### 类 2：IDE 状态注入

| type | 作用 |
|---|---|
| `selected_lines_in_ide` | 用户在 IDE 里选中的代码行（截断到 2000 字符） |
| `opened_file_in_ide` | 用户在 IDE 打开的文件 |
| `diagnostics` | IDE 的 lint/类型错误诊断 |

### 类 3：记忆与计划

| type | 作用 |
|---|---|
| `relevant_memories` | auto-memory surfacer 检索出的相关记忆（见 [prompt-injection 04](../prompt-injection/04-attachments与surfacer预算.md)） |
| `plan_file_reference` | plan mode 产出的计划文件 |
| `plan_mode` / `plan_mode_reentry` / `plan_mode_exit` | plan mode 阶段提示 |
| `auto_mode` / `auto_mode_exit` | auto mode 阶段提示 |

### 类 4：任务与技能提醒

| type | 作用 |
|---|---|
| `todo_reminder` | TodoWrite 提醒（V1）|
| `task_reminder` | TaskCreate/TaskUpdate 提醒（V2，gated）|
| `task_status` | 任务状态变化 |
| `invoked_skills` | 本会话已调用的 skill 指引 |
| `skill_listing` / `dynamic_skill` | 可用 skill 列表 |
| `verify_plan_reminder` | 验证计划提醒 |

### 类 5：协作（teammate / agent）

| type | 作用 |
|---|---|
| `team_context` | teammate 身份与团队资源（switch 前处理，gated `isAgentSwarmsEnabled`） |
| `teammate_mailbox` | teammate 间消息（switch 前处理） |
| `agent_mention` | `@agent-xxx` 提及 |

### 类 6：hook 与系统事件

| type | 作用 |
|---|---|
| `async_hook_response` | 异步 hook 的输出 |
| `hook_blocking_error` / `hook_success` / `hook_additional_context` / `hook_stopped_continuation` | 各类 hook 事件 |
| `critical_system_reminder` | 关键系统提醒 |
| `date_change` | 日期变更（如跨天）|
| `compaction_reminder` | 压缩发生提醒 |

### 类 7：资源声明与能力变化

| type | 作用 |
|---|---|
| `mcp_resource` / `mcp_instructions_delta` | MCP 资源 / 指令变化 |
| `deferred_tools_delta` | 延迟加载工具的变化 |
| `agent_listing_delta` | 可用 agent 列表变化 |
| `output_style` | 输出风格设定 |
| `token_usage` / `budget_usd` / `output_token_usage` | 用量统计 |
| `context_efficiency` / `ultrathink_effort` | 上下文效率 / thinking 力度 |
| `companion_intro` | companion 介绍 |
| `command_permissions` | 命令权限说明 |
| `queued_command` | 排队命令 |
| `skill_discovery` | skill 发现（switch 前处理，gated `EXPERIMENTAL_SKILL_SEARCH`） |

## 7.3 三种典型转换模式

50+ case 看起来多，但转换模式其实只有 3 种。

### 模式 A：伪造工具调用对

某些 attachment 表示"系统替模型读了个文件 / 列了个目录"。这种会展开成**伪造的 tool_use + tool_result 对**——让模型觉得"是我自己读的"。

`directory` case（`messages.ts:3525-3537`）：

```ts
case 'directory': {
  return wrapMessagesInSystemReminder([
    createToolUseMessage(BashTool.name, {
      command: `ls ${quote([attachment.path])}`,
      description: `Lists files in ${attachment.path}`,
    }),
    createToolResultMessage(BashTool, {
      stdout: attachment.content,
      stderr: '',
      interrupted: false,
    }),
  ])
}
```

展开成两条消息：

1. 一条**伪造的 assistant tool_use**——`ls <path>`；
2. 一条 tool_result——目录列表内容。

模型看到的就像"我刚才调了 Bash ls，得到了这个结果"。这种"伪造工具历史"的设计让 attachment 内容能自然融入模型的工作记忆——模型不会困惑"这个目录列表哪来的"。

`file` case（`messages.ts:3545-3591`）同理——伪造 FileRead 的 tool_use + tool_result。它有 4 个子类（按文件实际类型）：

```ts
case 'file': {
  const fileContent = attachment.content as FileReadToolOutput
  switch (fileContent.type) {
    case 'image': { /* 伪造 FileRead → image tool_result */ }
    case 'text': {
      return wrapMessagesInSystemReminder([
        createToolUseMessage(FileReadTool.name, { file_path: attachment.filename }),
        createToolResultMessage(FileReadTool, fileContent),
        ...(attachment.truncated
          ? [createUserMessage({
              content: `Note: The file ${attachment.filename} was too large and has been truncated to the first ${MAX_LINES_TO_READ} lines. Don't tell the user about this truncation. Use ${FileReadTool.name} to read more...`,
              isMeta: true,
            })]
          : []),
      ])
    }
    case 'notebook': { /* ... */ }
    case 'pdf': { /* ... */ }
  }
}
```

注意 text 子类多展开一条消息——如果文件被截断，加一条 isMeta 提示"被截断了，需要更多用 Read"。

### 模式 B：单条 isMeta user message

绝大多数 attachment 是"系统提醒"——展开成一条 `isMeta: true` 的 user message。

`edited_text_file` case（`messages.ts:3538-3544`）：

```ts
case 'edited_text_file':
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `Note: ${attachment.filename} was modified, either by the user or by a linter. This change was intentional, so make sure to take it into account as you proceed (ie. don't revert it unless the user asks you to). Don't tell the user this, since they are already aware. Here are the relevant changes (shown with line numbers):\n${attachment.snippet}`,
      isMeta: true,
    }),
  ])
```

`relevant_memories` case（`messages.ts:3708-3722`）：

```ts
case 'relevant_memories': {
  return wrapMessagesInSystemReminder(
    attachment.memories.map(m => {
      const header = m.header ?? memoryHeader(m.path, m.mtimeMs)
      return createUserMessage({
        content: `${header}\n\n${m.content}`,
        isMeta: true,
      })
    }),
  )
}
```

注意 `relevant_memories` 一个 attachment 展开成**多条** user message（每份记忆一条）。注释提到 header 用创建时存的、不重算——为了**prompt cache 命中**（重算 header 里有 mtime 等会变的东西，破坏 cache prefix）。

`todo_reminder` case（`messages.ts:3663-3679`）：

```ts
case 'todo_reminder': {
  const todoItems = attachment.content
    .map((todo, index) => `${index + 1}. [${todo.status}] ${todo.content}`)
    .join('\n')

  let message = `The TodoWrite tool hasn't been used recently. ...This is just a gentle reminder - ignore if not applicable. Make sure that you NEVER mention this reminder to the user\n`
  if (todoItems.length > 0) {
    message += `\n\nHere are the existing contents of your todo list:\n\n[${todoItems}]`
  }

  return wrapMessagesInSystemReminder([
    createUserMessage({ content: message, isMeta: true }),
  ])
}
```

注意提醒措辞里都有 **"NEVER mention this reminder to the user"**——这些是系统侧的"私语"，模型应该按它行动但不告诉用户。这是 isMeta 消息的典型特征。

### 模式 C：返回空数组（被忽略）

某些 attachment 在特定条件下不需要发给模型——返回 `[]`。

`dynamic_skill` case（`messages.ts:3723-3727`）：

```ts
case 'dynamic_skill': {
  // Dynamic skills are informational for the UI only - the skills themselves
  // are loaded separately and available via the Skill tool
  return []
}
```

`task_reminder` case 在 V2 未启用时（`messages.ts:3680-3683`）：

```ts
case 'task_reminder': {
  if (!isTodoV2Enabled()) {
    return []
  }
  // ...
}
```

`skill_listing` 在内容为空时（`messages.ts:3728-3731`）：

```ts
case 'skill_listing': {
  if (!attachment.content) {
    return []
  }
  // ...
}
```

返回 `[]` 让这个 attachment 在 normalize 后**完全消失**——不占 context、不影响模型。

## 7.4 switch 前的特殊处理（feature-gated）

有 4 种 attachment 类型**不在主 switch 里**，而在 switch 之前的 feature-gate 块里处理（`messages.ts:3456-3520`）。原因：case label 不能被 feature gate 包裹，但 if 块可以。

### `teammate_mailbox` / `team_context`（gated `isAgentSwarmsEnabled`）

```ts
if (isAgentSwarmsEnabled()) {
  if (attachment.type === 'teammate_mailbox') {
    return [createUserMessage({
      content: getTeammateMailbox().formatTeammateMessages(attachment.messages),
      isMeta: true,
    })]
  }
  if (attachment.type === 'team_context') {
    return [createUserMessage({
      content: `<system-reminder>\n# Team Coordination\n...`,
      isMeta: true,
    })]
  }
}
```

teammate / 团队相关——只在 agent swarms 功能启用时才处理。

### `skill_discovery`（gated `EXPERIMENTAL_SKILL_SEARCH`）

```ts
if (feature('EXPERIMENTAL_SKILL_SEARCH')) {
  if (attachment.type === 'skill_discovery') {
    if (attachment.skills.length === 0) return []
    const lines = attachment.skills.map(s => `- ${s.name}: ${s.description}`)
    return wrapMessagesInSystemReminder([
      createUserMessage({
        content: `Skills relevant to your task:\n\n${lines.join('\n')}\n\n` +
          `These skills encode project-specific conventions. ` +
          `Invoke via Skill("<name>") for complete instructions.`,
        isMeta: true,
      }),
    ])
  }
}
```

注释（`messages.ts:3503-3505`）解释了这个模式：

> skill_discovery handled here (not in the switch) so the 'skill_discovery' string literal lives inside a feature()-guarded block. A case label can't be gated, but this pattern can — same approach as teammate_mailbox above.

这是个 TypeScript / bundler 的实务技巧：`feature()` 是编译期 dead-code elimination 的标记，被 gate 的 `if` 块在功能关闭时整段被移除（连同 `'skill_discovery'` 字符串常量）。但 `case 'skill_discovery':` 这种 label 没法被 dead-code elimination 干净移除。所以**故意把这几个 case 提到 switch 外**用 if 块写。

switch 主体上有个 biome-ignore 注释（`messages.ts:3522-3523`）声明这几个类型"已在上面处理"，避免 exhaustiveness check 报错。

## 7.5 attachment normalizer 的几个设计判断

### 1. 伪造工具历史让内容自然融入

`directory` / `file` 不是简单塞一段文本——而是伪造完整的 tool_use + tool_result 对。模型看到的是"自己读过的"，不需要理解"这段内容是哪来的"。这降低了模型的认知负担。

### 2. 几乎所有都是 isMeta

除了模式 A 的伪造工具对（它们是真正的 assistant/user 消息形态），模式 B 的提醒全是 `isMeta: true`。这让它们在 UI 隐藏、但模型能看到。

### 3. 措辞里反复出现"don't tell the user"

`edited_text_file`、`file`(truncated)、`todo_reminder`——多处明说"Don't tell the user / NEVER mention this reminder"。这是系统侧"私语"的统一处理：模型按它行动，但不向用户暴露这些系统机制的存在。

### 4. 与 prompt cache 的协同

`relevant_memories` 用存储的 header 而非重算——为了 cache prefix 稳定。这是 attachment normalizer 里隐藏的"性能-正确性"权衡：normalize 必须保证同样的 attachment 每轮渲染出**字节相同**的内容，否则 prompt cache 全 miss。

### 5. 空数组让 attachment 优雅消失

模式 C 的 `return []`——某个 attachment 在当前条件下不需要时直接返回空。比"塞一个空消息再过滤"干净。

### 6. feature-gated 类型提到 switch 外

为了配合 bundler 的 dead-code elimination——这是 TypeScript + bun:bundle 环境特有的实务技巧。

## 7.6 与 prompt-injection 专题的呼应

attachment 全是"友方注入"——系统主动塞进上下文的内容。对照 [prompt-injection 04](../prompt-injection/04-attachments与surfacer预算.md)：

- **本篇（07）**：讲 attachment **怎么转成 user message**——格式层；
- **prompt-injection 04**：讲 attachment **被多少配额约束**——治理层。

两篇合起来是友方注入的完整画面：

```
attachment 产生 (surfacer / reminder 触发器 / IDE 事件 / hook ...)
       │
       ▼
配额检查 (prompt-injection 04: 频率 5-10 turn / 单次 4KB / 会话 60KB)
       │
       ▼ 通过
normalizeAttachmentForAPI (本篇: 50+ case → UserMessage[])
       │
       ▼
ensureSystemReminderWrap (加 <system-reminder> 标签)
       │
       ▼
合并进前一条 user message
       │
       ▼
smooshSystemReminderSiblings (折进 tool_result)
       │
       ▼
发 API
```

`<system-reminder>` 标签机制本身见 [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md)。

## 7.7 把 attachment 类型当"系统能说的话"目录学

最后一个视角：50+ attachment 类型其实是一份**"Claude Code 系统能向模型说哪些话"的完整清单**。

如果你想理解"Claude Code 在用户输入之外，还往模型上下文里塞了什么"，看这个 switch 就够了：

- **替模型读文件**（file / directory / nested_memory）
- **告诉模型环境变了**（edited_text_file / date_change / diagnostics）
- **提醒模型用工具**（todo_reminder / task_reminder / verify_plan_reminder）
- **给模型补充上下文**（relevant_memories / selected_lines_in_ide / invoked_skills）
- **协调多代理**（team_context / teammate_mailbox / agent_mention）
- **声明能力变化**（deferred_tools_delta / agent_listing_delta / mcp_instructions_delta）
- **传递 hook 结果**（async_hook_response / hook_* 系列）
- **报告用量**（token_usage / budget_usd / context_efficiency）

每加一种"系统想对模型说的话"，就在这个 switch 里加一个 case。这个 switch 的增长史，某种程度上就是 Claude Code 这个 agent "系统侧表达能力"的演化史。

## 7.8 小结

- `normalizeAttachmentForAPI` 是 50+ case 的超长 switch，把每种 attachment 转成 `UserMessage[]`；
- 转换模式只有 3 种：伪造工具调用对（A）/ 单条 isMeta user message（B）/ 返回空数组（C）；
- 4 个 feature-gated 类型提到 switch 外用 if 块——配合 bundler dead-code elimination；
- 伪造工具历史让 attachment 内容自然融入模型工作记忆；
- 几乎全是 isMeta，措辞里反复"don't tell the user"——系统侧私语；
- `relevant_memories` 用存储 header 而非重算——保 prompt cache 稳定；
- 这个 switch 本质是一份"系统能向模型说哪些话"的完整目录。

下一篇 → [08 端到端模拟与 feature gate 时间线](./08-端到端模拟与feature-gate时间线.md)
