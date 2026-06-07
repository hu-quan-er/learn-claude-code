# Skill 与 Tool 上下文保留策略

> Claude Code 上下文管理分析系列 - 第8篇
> 源码版本：claude-code 2.1.88

---

## 概述

历史加载过的 skill、tool、tool result 并不会以同一种方式“永久留在上下文里”。

Claude Code 把“上下文”拆成至少四层：

1. **消息历史**：已经注入到 conversation 的 user / assistant / attachment / system message。
2. **运行时状态**：`ToolUseContext`、`AppState`、module-level cache、`STATE.invokedSkills`。
3. **API 请求参数**：每轮重新构建的 `system`、`messages`、`tools[]`、betas、cache_control。
4. **压缩恢复材料**：compact boundary、summary、post-compact attachments、content replacement records。

因此，判断“会不会一直保存”必须先问：保存在哪一层、跨不跨 turn、跨不跨 compact、跨不跨 resume。

---

## 1. 总结论

| 对象 | 是否进入消息历史 | 是否跨 compact | 是否跨 resume | 保留策略 |
|------|------------------|----------------|---------------|----------|
| `skill_listing` | 是，作为 attachment 展开成 system-reminder | 不主动重发 | 已有 listing 会抑制重复注入 | 只用于曝光可用 skill，按 agent 去重 |
| inline skill 内容 | 是，作为 meta user message | 是，通过 `invoked_skills` 恢复 | 是，从 `invoked_skills` attachment 重建 | 单独的 `STATE.invokedSkills` 持久化策略 |
| fork skill 内容 | 进入子 agent 上下文 | 只在对应 agent 维度恢复 | 主会话不恢复子 agent skill | `agentId` 隔离，子任务结束后清理 |
| skill `allowedTools` | 不作为长期文本保存 | 否 | 否 | `contextModifier` / `command_permissions` 临时放行 |
| skill `model` / `effort` | 不作为长期文本保存 | 否 | 否 | 当前 query 链或当前 turn 的运行时修改 |
| 普通 tool schema | 不进入消息历史 | 每轮重建 | 每轮重建 | 来自当前工具池、权限、MCP 状态 |
| ToolSearch 发现的 deferred tool | `tool_reference` 在 tool_result 中出现 | 是，compact boundary 携带工具名 | 随消息/边界恢复 | 保存“已发现工具名”，不是完整 schema 文本 |
| tool result | 是，作为 tool_result | 可能被摘要/清理/替换 | 替换记录可恢复 | 大结果落盘、聚合预算、microcompact/cache edits |

核心结论：

- **skill 内容有专门的 compact 保留策略**：`invokedSkills`。
- **tool result 有专门的瘦身策略**：持久化、内容替换、microcompact、cache edits。
- **tool schema 没有按历史永久保存**：每轮 API 请求按当前工具池重建。
- **skill 权限、模型、effort 是运行时副作用，不是长期上下文材料**。

---

## 2. `skill_listing`：曝光列表，不是长期知识库

`skill_listing` 来自 attachment 系统。

源码位置：

- `restored-src/src/utils/attachments.ts`
- `restored-src/src/utils/messages.ts`

生成入口：

```ts
// utils/attachments.ts
const sentSkillNames = new Map<string, Set<string>>()

async function getSkillListingAttachments(toolUseContext) {
  if (!toolUseContext.options.tools.some(t => toolMatchesName(t, SKILL_TOOL_NAME))) {
    return []
  }

  const agentKey = toolUseContext.agentId ?? ''
  let sent = sentSkillNames.get(agentKey)
  ...
  const newSkills = allCommands.filter(cmd => !sent.has(cmd.name))
  ...
  return [{
    type: 'skill_listing',
    content,
    skillCount: newSkills.length,
    isInitial,
  }]
}
```

展开给模型时变成：

```ts
// utils/messages.ts
case 'skill_listing':
  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `The following skills are available for use with the Skill tool:\n\n${attachment.content}`,
      isMeta: true,
    }),
  ])
```

这里有三个关键点。

### 2.1 按 agent 去重

`sentSkillNames` 的 key 是 `agentId ?? ''`。

主会话和子 agent 各自有自己的已发送 skill 集合，避免主会话发过 listing 后，子 agent turn-0 看不到任何 skill。

### 2.2 compact 后不主动重发

`compact.ts` 和 `postCompactCleanup.ts` 都明确注释：compact 后不 reset `sentSkillNames`。

原因是完整 `skill_listing` 可能带来几千 token 的 cache creation 成本，而已实际调用过的 skill 会通过 `invoked_skills` 单独恢复。

```ts
// compact.ts
// Intentionally NOT resetting sentSkillNames: re-injecting the full
// skill_listing (~4K tokens) post-compact is pure cache_creation ...
// The model still has SkillTool in its schema and invoked_skills attachment
// preserves used-skill content.
```

### 2.3 resume 时也避免重复 listing

`conversationRecovery.ts` 在恢复历史时，如果发现历史中已经有 `skill_listing` attachment，会调用 `suppressNextSkillListing()`。

这说明 `skill_listing` 不是“每次启动都重放的长期状态”，而是一次性的能力曝光提示。

---

## 3. 已执行 skill 内容：`invokedSkills` 单独保留

inline skill 被调用时，真正注入模型的是 skill 展开后的 prompt 内容。

用户 `/skill` 和模型 `SkillTool` 最终都会走：

- `restored-src/src/utils/processUserInput/processSlashCommand.tsx`
- `restored-src/src/tools/SkillTool/SkillTool.ts`

公共落点是 `processPromptSlashCommand()` / `getMessagesForPromptSlashCommand()`。

关键代码：

```ts
// processSlashCommand.tsx
const result = await command.getPromptForCommand(args, context)
const skillPath = command.source ? `${command.source}:${command.name}` : command.name
const skillContent = result
  .filter((b): b is TextBlockParam => b.type === 'text')
  .map(b => b.text)
  .join('\n\n')

addInvokedSkill(
  command.name,
  skillPath,
  skillContent,
  getAgentContext()?.agentId ?? null,
)
```

`addInvokedSkill()` 写入 module-level bootstrap state：

```ts
// bootstrap/state.ts
export type InvokedSkillInfo = {
  skillName: string
  skillPath: string
  content: string
  invokedAt: number
  agentId: string | null
}

export function addInvokedSkill(skillName, skillPath, content, agentId = null) {
  const key = `${agentId ?? ''}:${skillName}`
  STATE.invokedSkills.set(key, {
    skillName,
    skillPath,
    content,
    invokedAt: Date.now(),
    agentId,
  })
}
```

这张表就是 skill 内容跨 compact 的单独策略。

---

## 4. compact 后如何恢复 skill

全量 compact 会把旧消息压成 summary。原来注入到消息历史里的 skill prompt 可能被摘要掉，所以 compact 会额外生成一个 `invoked_skills` attachment。

源码位置：

- `restored-src/src/services/compact/compact.ts`
- `restored-src/src/utils/messages.ts`

预算常量：

```ts
export const POST_COMPACT_MAX_TOKENS_PER_SKILL = 5_000
export const POST_COMPACT_SKILLS_TOKEN_BUDGET = 25_000
```

恢复 attachment 生成：

```ts
export function createSkillAttachmentIfNeeded(agentId?: string): AttachmentMessage | null {
  const invokedSkills = getInvokedSkillsForAgent(agentId)
  if (invokedSkills.size === 0) return null

  let usedTokens = 0
  const skills = Array.from(invokedSkills.values())
    .sort((a, b) => b.invokedAt - a.invokedAt)
    .map(skill => ({
      name: skill.skillName,
      path: skill.skillPath,
      content: truncateToTokens(skill.content, POST_COMPACT_MAX_TOKENS_PER_SKILL),
    }))
    .filter(skill => {
      const tokens = roughTokenCountEstimation(skill.content)
      if (usedTokens + tokens > POST_COMPACT_SKILLS_TOKEN_BUDGET) return false
      usedTokens += tokens
      return true
    })

  return createAttachmentMessage({
    type: 'invoked_skills',
    skills,
  })
}
```

展开给模型：

```ts
case 'invoked_skills':
  const skillsContent = attachment.skills
    .map(skill => `### Skill: ${skill.name}\nPath: ${skill.path}\n\n${skill.content}`)
    .join('\n\n---\n\n')

  return wrapMessagesInSystemReminder([
    createUserMessage({
      content: `The following skills were invoked in this session. Continue to follow these guidelines:\n\n${skillsContent}`,
      isMeta: true,
    }),
  ])
```

这意味着：

- compact summary 不负责完整保存 skill 规则；
- `invoked_skills` attachment 负责把已用 skill 的关键内容重新注入；
- 预算压力下按最近使用优先，且每个 skill 只保留头部 5K tokens；
- 超出 25K 总预算的旧 skill 会被丢弃。

---

## 5. post-compact 为什么不清 `invokedSkills`

`postCompactCleanup.ts` 里有一段非常关键的注释：

```ts
// Note: We intentionally do NOT clear invoked skill content here.
// Skill content must survive across multiple compactions so that
// createSkillAttachmentIfNeeded() can include the full skill text
// in subsequent compaction attachments.
```

这说明 `invokedSkills` 的生命周期不是“一次 compact 后清空”，而是跨多次 compact 保留。

但是它也不是永久的：

- `/clear caches` 会调用 `clearInvokedSkills(preservedAgentIds)`；
- `/clear conversation` 会清 session caches，但保留仍在运行的 background agent 对应状态；
- fork skill 的 agent scope 会在子任务结束后清理；
- 新进程 resume 需要从历史中的 `invoked_skills` attachment 重建。

resume 重建入口：

```ts
// conversationRecovery.ts
export function restoreSkillStateFromMessages(messages: Message[]): void {
  for (const message of messages) {
    if (message.type !== 'attachment') continue
    if (message.attachment.type === 'invoked_skills') {
      for (const skill of message.attachment.skills) {
        addInvokedSkill(skill.name, skill.path, skill.content, null)
      }
    }
  }
}
```

所以准确说法是：

> 已执行 skill 的内容会在当前 session state 中保留，并通过 `invoked_skills` attachment 跨 compact / resume 恢复；但它受预算、agent scope 和 clear 操作限制，不是无条件永久保存。

---

## 6. agent 隔离：fork skill 不污染主会话

`invokedSkills` 的 key 是 `(agentId, skillName)`。

```ts
const key = `${agentId ?? ''}:${skillName}`
```

compact 时只取当前 agent 对应的 skills：

```ts
const invokedSkills = getInvokedSkillsForAgent(agentId)
```

这解决两个问题：

1. 主会话调用过的 skill 不会自动灌给无关子 agent。
2. 子 agent 内部调用过的 skill 不会在主会话 compact 后复活。

fork skill 的模型调用路径里还会显式清理子 agent skill 状态：

```ts
// SkillTool.ts
const agentId = createAgentId()
...
try {
  for await (const message of runAgent({ override: { agentId }, ... })) {
    ...
  }
} finally {
  clearInvokedSkillsForAgent(agentId)
}
```

这和 AgentTool 的核心价值一致：子 agent 内部的工具噪音、文件读取、skill 调用不直接污染父上下文，父上下文只看到最终结果。

---

## 7. `allowedTools` / `model` / `effort` 不是长期上下文

skill frontmatter 可以声明：

```yaml
allowed-tools: [Read, Grep, Bash(git:*)]
model: opus
effort: high
```

这些字段不是通过 `invoked_skills` 长期恢复的内容，而是执行期策略。

### 7.1 inline skill

`SkillTool` inline 路径返回 `contextModifier()`：

```ts
return {
  data: { success: true, commandName, allowedTools, model },
  newMessages,
  contextModifier(ctx) {
    let modifiedContext = ctx

    if (allowedTools.length > 0) {
      modifiedContext = {
        ...modifiedContext,
        getAppState() {
          const appState = previousGetAppState()
          return {
            ...appState,
            toolPermissionContext: {
              ...appState.toolPermissionContext,
              alwaysAllowRules: {
                ...appState.toolPermissionContext.alwaysAllowRules,
                command: [...new Set([...oldRules, ...allowedTools])],
              },
            },
          }
        },
      }
    }

    if (model) {
      modifiedContext = {
        ...modifiedContext,
        options: {
          ...modifiedContext.options,
          mainLoopModel: resolveSkillModelOverride(model, ctx.options.mainLoopModel),
        },
      }
    }

    if (effort !== undefined) {
      const previousGetAppState = modifiedContext.getAppState
      modifiedContext = {
        ...modifiedContext,
        getAppState() {
          const appState = previousGetAppState()
          return { ...appState, effortValue: effort }
        },
      }
    }

    return modifiedContext
  },
}
```

执行器会把这个 modifier 应用到后续工具循环。

但 REPL 每次用户输入都会重新构造 `ToolUseContext`：

```ts
const toolUseContext = getToolUseContext(
  messagesIncludingNewMessages,
  newMessages,
  abortController,
  mainLoopModelParam,
)
```

所以这类修改是当前 query 链 / 当前 turn 的运行时副作用，不是会话级永久权限。

### 7.2 fork skill

fork 路径通过 `prepareForkedCommandContext()` 把 `allowedTools` 放进子 agent 的 permission context：

```ts
const allowedTools = parseToolListFromCLI(command.allowedTools ?? [])
const modifiedGetAppState = createGetAppStateWithAllowedTools(
  context.getAppState,
  allowedTools,
)
```

这只作用于 fork 子 agent 的生命周期。子 agent 跑完，这个权限上下文也随之结束。

---

## 8. tool schema：每轮重建，不靠历史保存

工具定义不是像 skill 内容那样作为历史消息保存。

工具池来自：

- `restored-src/src/tools.ts`
- `restored-src/src/services/api/claude.ts`
- `restored-src/src/tools/ToolSearchTool/`

核心组合函数：

```ts
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

每次 API 请求前，`claude.ts` 会根据 ToolSearch 是否启用过滤工具：

```ts
if (useToolSearch) {
  const discoveredToolNames = extractDiscoveredToolNames(messages)
  filteredTools = tools.filter(tool => {
    if (!deferredToolNames.has(tool.name)) return true
    if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true
    return discoveredToolNames.has(tool.name)
  })
} else {
  filteredTools = tools.filter(t => !toolMatchesName(t, TOOL_SEARCH_TOOL_NAME))
}
```

也就是说：

- 普通内置工具、非 deferred 工具每轮从当前工具池进入 `tools[]`；
- MCP/deferred 工具不一定一开始全量进入；
- deny rules、MCP 连接状态、feature flag 都可能改变本轮 `tools[]`；
- 历史里不会保存“之前所有工具 schema 的完整文本”。

---

## 9. ToolSearch：保存的是已发现工具名

ToolSearchTool 的结果不是普通文本，而是 `tool_reference`：

```ts
// ToolSearchTool.ts
return {
  type: 'tool_result',
  tool_use_id: toolUseID,
  content: content.matches.map(name => ({
    type: 'tool_reference',
    tool_name: name,
  })),
}
```

后续请求用 `extractDiscoveredToolNames(messages)` 扫描历史里的 `tool_reference`，决定哪些 deferred tool schema 应该进入 `tools[]`。

compact 会把这份“已发现工具名集合”写到 compact boundary：

```ts
const preCompactDiscovered = extractDiscoveredToolNames(messages)
if (preCompactDiscovered.size > 0) {
  boundaryMarker.compactMetadata.preCompactDiscoveredTools = [
    ...preCompactDiscovered,
  ].sort()
}
```

恢复时再从 boundary 读回：

```ts
if (msg.type === 'system' && msg.subtype === 'compact_boundary') {
  const carried = msg.compactMetadata?.preCompactDiscoveredTools
  if (carried) {
    for (const name of carried) discoveredTools.add(name)
  }
}
```

这是一套独立于 skill 的工具发现持久化策略。

它保存的是：

- 哪些 deferred tools 曾经被 ToolSearch 发现；
- compact 后继续把这些工具 schema 放进 API 请求。

它不保存：

- 工具 schema 的完整历史文本；
- 已断开 MCP server 的不可用工具引用。

`normalizeMessagesForAPI()` 还会过滤不可用 `tool_reference`，防止 API 报 `Tool reference not found in available tools`。

---

## 10. tool result：进入历史，但不保证原文常驻

tool result 会进入消息历史：

```text
assistant: tool_use
user: tool_result
```

但它有多套瘦身策略。

### 10.1 单工具大结果持久化

默认阈值是 50,000 字符：

```ts
export const DEFAULT_MAX_RESULT_SIZE_CHARS = 50_000
```

超过阈值后，完整结果写入磁盘，模型只看到路径和预览：

```xml
<persisted-output>
Output too large (...). Full output saved to: ...

Preview:
...
</persisted-output>
```

`maxResultSizeChars = Infinity` 的工具例外，比如 Read。源码里明确注释：Read 结果再写文件让模型重新 Read 是循环，所以跳过持久化。

### 10.2 每消息聚合预算

一轮并发工具可能每个结果都没超过单工具阈值，但合起来非常大。

`enforceToolResultBudget()` 用 `tool_use_id` 做三态跟踪：

| 状态 | 含义 | 行为 |
|------|------|------|
| `fresh` | 新结果 | 可被选择持久化 |
| `frozen` | 已见过且未替换 | 永远不再替换，保护 prompt cache |
| `mustReapply` | 已替换过 | 每轮重放相同 replacement |

这说明旧 tool result 的命运一旦决定，就会被冻结，避免后续 turn 改写历史前缀导致缓存失效。

### 10.3 Microcompact / Cached Microcompact

系统提示里也会提醒模型：

```text
Old tool results will be automatically cleared from context to free up space.
The N most recent results are always kept.
```

这是针对旧工具结果的另一层策略：

- microcompact 可以把旧结果替换成 `[Old tool result content cleared]`；
- cached microcompact 可以通过 API `cache_edits` 删除缓存中的旧 tool result；
- time-based microcompact 会在时间间隔较长时清理旧工具结果，避免缓存 TTL 过期后重发巨量历史。

因此，tool result 是“会进入历史，但会被预算系统和压缩系统持续瘦身”的上下文材料。

---

## 11. compact 前还会剥离会重新注入的内容

`compact.ts` 有 `stripReinjectedAttachments()`：

```ts
export function stripReinjectedAttachments(messages: Message[]): Message[] {
  if (feature('EXPERIMENTAL_SKILL_SEARCH')) {
    return messages.filter(
      m =>
        !(
          m.type === 'attachment' &&
          (m.attachment.type === 'skill_discovery' ||
            m.attachment.type === 'skill_listing')
        ),
    )
  }
  return messages
}
```

这体现了一个重要设计原则：

> 会被系统重新生成的提示，不应该交给 summarizer 总结，否则会浪费 token，还可能把过期建议写进摘要。

`skill_listing` 和 `skill_discovery` 属于能力曝光层，不属于需要被摘要保存的任务事实。

---

## 12. 端到端时间线

### 12.1 inline skill

```text
Round N:
  assistant -> tool_use Skill("verify")
  SkillTool:
    getPromptForCommand()
    addInvokedSkill(name, path, content, agentId=null)
    newMessages += skill content as meta user message
    contextModifier += allowedTools/model/effort
  query recursion:
    model sees skill content
    model may call Read/Bash/Edit...

Later compact:
  old messages -> summary
  createSkillAttachmentIfNeeded(agentId=null)
  postCompact messages include invoked_skills attachment

Resume:
  restoreSkillStateFromMessages()
  addInvokedSkill(...) rebuilds STATE.invokedSkills
```

### 12.2 fork skill

```text
Round N:
  assistant -> tool_use Skill("large-refactor")
  SkillTool:
    createAgentId()
    prepareForkedCommandContext()
    runAgent({ override: { agentId } })

Inside child agent:
  child may invoke more skills/tools
  addInvokedSkill(..., agentId=<child>)

Child finishes:
  parent receives compact result text
  clearInvokedSkillsForAgent(childAgentId)

Parent compact:
  only agentId=null skills restored
```

### 12.3 ToolSearch

```text
Round N:
  tools[] includes non-deferred tools + ToolSearch
  assistant -> tool_use ToolSearch("slack")
  user -> tool_result [tool_reference: mcp__slack__send_message]

Round N+1:
  extractDiscoveredToolNames(messages)
  filteredTools includes mcp__slack__send_message schema

compact:
  summary drops original tool_reference message
  compact boundary stores preCompactDiscoveredTools

Round after compact:
  extractDiscoveredToolNames(boundary)
  same deferred tool remains loaded
```

---

## 13. 设计取舍

### 13.1 为什么 skill 内容要单独存

skill prompt 经常是操作规程，而不是对话事实。

如果只依赖 compact summary：

- summarizer 可能把细节压掉；
- hooks、allowed-tools、引用路径等关键规则可能丢失；
- 多次 compact 后规则会逐渐漂移。

所以系统用 `invokedSkills` 保存原始展开文本，再用 `invoked_skills` attachment 恢复。

### 13.2 为什么 skill listing 不同样保存

`skill_listing` 是“有哪些 skill 可用”的索引，不是“当前任务已经采用了哪些规则”。

保存全部 listing 的代价高，而且很多条目与当前任务无关。

因此：

- 可用列表按需曝光；
- 已使用内容单独恢复；
- 新增 skill 通过动态发现和 reset cache 再曝光。

### 13.3 为什么 tool schema 不进历史

tool schema 是 API 请求参数，不是 conversation message。

把所有历史工具定义写进消息会带来三个问题：

- MCP 工具很多时 token 爆炸；
- MCP server 断开后历史 schema 会变脏；
- prompt cache 会频繁被动态工具扰动。

ToolSearch 的做法是只把“发现过哪些 deferred tools”留在历史/compact boundary 中，然后每轮用当前可用工具池重新构建 schema。

### 13.4 为什么 tool result 要冻结替换决策

如果一个旧 tool result 今天保留原文，明天又被替换成预览，缓存前缀就会变化。

所以 `toolResultStorage.ts` 选择：

- 第一次见到时决定命运；
- 已保留的永不替换；
- 已替换的每轮重放完全相同 replacement；
- 新结果只影响新尾部。

这是一种用空间换 prompt cache 稳定性的设计。

---

## 14. 与现有文档的关系

这篇补的是横向问题：“历史加载过的 skill/tool 是否一直保留？”

相关章节分布如下：

| 主题 | 已有文档 |
|------|----------|
| skill 装载、列表、动态发现 | `topics/skill/03-装载层逐行精读.md`、`topics/skill/04-命令聚合总线.md`、`topics/skill/07-动态发现与条件激活.md` |
| 用户 `/skill` 和 `addInvokedSkill` | `topics/skill/05-用户调用链路追踪.md` |
| 模型 `SkillTool`、fork、contextModifier | `topics/skill/06-模型调用链路追踪.md` |
| fork skill 的 agent 隔离 | `topics/skill/10-常见疑惑与陷阱.md` |
| tool result 预算、compact 策略 | `topics/context-management/02-上下文压缩与自动压缩机制.md` |
| prompt cache 与 content replacement | `topics/context-management/03-提示缓存与内容替换机制.md` |
| tool_reference normalize | `topics/messages-pipeline/03-normalize主循环与三路分发.md` |

本篇把这些分散机制串成一个保留策略矩阵。

---

## 15. 源码地图

| 文件 | 关键点 |
|------|--------|
| `src/bootstrap/state.ts` | `STATE.invokedSkills`、`addInvokedSkill()`、`clearInvokedSkills()` |
| `src/tools/SkillTool/SkillTool.ts` | inline/fork 分流、`contextModifier`、远程 skill 注入 |
| `src/utils/processUserInput/processSlashCommand.tsx` | `processPromptSlashCommand()`、skill 内容注入、`addInvokedSkill()` |
| `src/utils/forkedAgent.ts` | fork skill 的 `allowedTools` 注入 |
| `src/services/compact/compact.ts` | `createSkillAttachmentIfNeeded()`、`invoked_skills`、compact boundary 工具名携带 |
| `src/services/compact/postCompactCleanup.ts` | compact 后不清 `invokedSkills` |
| `src/utils/conversationRecovery.ts` | resume 后从 `invoked_skills` 恢复 state |
| `src/utils/attachments.ts` | `skill_listing` 去重、suppress、按 agent 分组 |
| `src/utils/messages.ts` | attachment 展开、`tool_reference` 过滤、API normalize |
| `src/tools.ts` | `assembleToolPool()` 工具池重建 |
| `src/services/api/claude.ts` | `filteredTools`、ToolSearch 动态工具 schema |
| `src/tools/ToolSearchTool/ToolSearchTool.ts` | `tool_reference` 结果 |
| `src/utils/toolSearch.ts` | `extractDiscoveredToolNames()` |
| `src/utils/toolResultStorage.ts` | 大 tool result 持久化、三态替换 |

---

## 16. 一句话结论

Claude Code 没有把“历史加载过的所有 skill/tool”无脑塞进上下文长期保存。

它的策略是：

> 已使用 skill 的内容单独保留并跨 compact 恢复；可用 skill 列表只按需曝光；工具 schema 每轮从当前工具池重建；ToolSearch 只持久化已发现工具名；工具结果进入历史但持续受持久化、替换和压缩策略瘦身。
