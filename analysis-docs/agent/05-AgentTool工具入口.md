# 05 - AgentTool 工具入口

## 文件定位

`src/tools/AgentTool/AgentTool.tsx`（1397 LOC）是把 `Agent` 这个 tool 暴露给模型的"外壳"。它做四件事：

1. 用 Zod 定义 input/output schema，按 feature gate 裁剪字段
2. 计算发给模型的 `tool.prompt()`（拼装可派发 agent 列表）
3. 在 `call()` 中分流到 02 篇说的 5 种派发模式
4. 把 5 种派发结果统一映射成 `tool_result` 文本

工具本身的内部实现（fork 父状态、内部 query 循环）在 `runAgent.ts`，参见 06 篇。

## 输入 schema 的两层组合

```typescript
// AgentTool.tsx:82-88 - 基础字段
const baseInputSchema = lazySchema(() => z.object({
  description: z.string().describe('A short (3-5 word) description of the task'),
  prompt: z.string().describe('The task for the agent to perform'),
  subagent_type: z.string().optional().describe('The type of specialized agent to use for this task'),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional().describe(
    "Optional model override for this agent. Takes precedence over the agent definition's model frontmatter. If omitted, uses the agent definition's model, or inherits from the parent."
  ),
  run_in_background: z.boolean().optional().describe(
    'Set to true to run this agent in the background. You will be notified when it completes.'
  )
}))

// AgentTool.tsx:91-102 - 完整字段 = 基础 + multi-agent + isolation/cwd
const fullInputSchema = lazySchema(() => {
  const multiAgentInputSchema = z.object({
    name: z.string().optional().describe(...),
    team_name: z.string().optional().describe(...),
    mode: permissionModeSchema().optional().describe(...)
  })
  return baseInputSchema().merge(multiAgentInputSchema).extend({
    isolation: ("external" === 'ant'
      ? z.enum(['worktree', 'remote'])
      : z.enum(['worktree'])).optional().describe(...),
    cwd: z.string().optional().describe(
      'Absolute path to run the agent in. Overrides the working directory for all filesystem and shell operations within this agent. Mutually exclusive with isolation: "worktree".'
    )
  })
})

// AgentTool.tsx:110-125 - 按 feature 裁剪
export const inputSchema = lazySchema(() => {
  const schema = feature('KAIROS') ? fullInputSchema() : fullInputSchema().omit({ cwd: true })
  return isBackgroundTasksDisabled || isForkSubagentEnabled()
    ? schema.omit({ run_in_background: true })
    : schema
})
```

### 字段总览

| 字段 | 类型 | 触发什么 | 哪些情况下 schema 砍掉 |
|------|------|----------|-----------------------|
| `description` | string，3-5 词 | UI 显示、analytics tag | 永远在 |
| `prompt` | string | 子代理首轮 user message | 永远在 |
| `subagent_type` | string optional | 选择 agent 定义；省略 → fork（启用时）或 general-purpose（默认） | 永远在 |
| `model` | `'sonnet'\|'opus'\|'haiku'` optional | 04 篇的第 2 级优先级 | 永远在 |
| `run_in_background` | boolean optional | async_launched 派发模式 | `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS` 或 fork 启用时 |
| `name` | string optional | swarm teammate 名 | 永远在（但 `isAgentSwarmsEnabled()` 决定能否真用） |
| `team_name` | string optional | swarm team 名，与 `name` 配对 | 同上 |
| `mode` | PermissionMode optional | 派发 teammate 时强制权限模式 | 同上 |
| `isolation` | `'worktree'` 或 ant 多 `'remote'` | worktree/remote 隔离 | `"remote"` 仅 ant 可见 |
| `cwd` | string optional | KAIROS 模式下的 cwd 覆盖；与 worktree 互斥 | 非 KAIROS 时砍掉 |

### 几条隐式约束

- `team_name + name` 同传 = teammate 派发；缺一不进入 swarm 分支。
- `isolation: "worktree"` 与 `cwd` 互斥（描述里写明：mutually exclusive）。
- `run_in_background` 在 fork 启用时被 schema 移除，因为 fork 启用时所有 spawn 强制 async。
- `subagent_type` 在 fork 实验启用时是 optional（默认走 fork），关闭时也是 optional 但默认走 `general-purpose`。

## prompt 字段的拼装

`AgentTool` 的 `prompt(...)` 函数（`AgentTool.tsx:197-225`）：

```typescript
async prompt({ agents, tools, getToolPermissionContext, allowedAgentTypes }) {
  const toolPermissionContext = await getToolPermissionContext()

  // 收集已有工具的 MCP server 名（用于按 requiredMcpServers 过滤 agent）
  const mcpServersWithTools: string[] = []
  for (const tool of tools) {
    if (tool.name?.startsWith('mcp__')) {
      const serverName = tool.name.split('__')[1]
      if (serverName && !mcpServersWithTools.includes(serverName)) mcpServersWithTools.push(serverName)
    }
  }

  // 双层过滤：MCP 要求 + permission deny
  const agentsWithMcpRequirementsMet = filterAgentsByMcpRequirements(agents, mcpServersWithTools)
  const filteredAgents = filterDeniedAgents(agentsWithMcpRequirementsMet, toolPermissionContext, AGENT_TOOL_NAME)

  const isCoordinator = feature('COORDINATOR_MODE') ? isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE) : false
  return await getPrompt(filteredAgents, isCoordinator, allowedAgentTypes)
}
```

注意：**模型看到的 description = `tool.prompt()` 的返回值**（TodoList 06 篇里讲过的事实，这里再次出现）。`tool.description()` 字面是 `'Launch a new agent'`，但那是 ToolSearch / UI 用的短摘要，**不进 API 请求体**。

`getPrompt()` 是 `prompt.ts` 里的拼装函数：

```text
shared base
  "Launch a new agent to handle complex, multi-step tasks autonomously."
  "The Agent tool launches specialized agents (subprocesses) that ..."
  Available agent types and the tools they have access to:
  - general-purpose: ... (Tools: All tools)
  - Explore: ... (Tools: All tools except Agent, ExitPlanMode, Edit, Write, NotebookEdit)
  - ... (来自 filteredAgents 的 formatAgentLine)
  When using the Agent tool, specify a subagent_type ...

isCoordinator ?
  → 只返回上面 shared 段（coordinator 自己的 system prompt 覆盖其它）

非 coordinator:
  + "When NOT to use" 段
  + Usage notes（并发提示、background 提示、SendMessage 提示、worktree 提示、teammate 限制等）
  + Writing the prompt 段
  + Examples 段（fork 启用时给 fork 的例子，否则给 test-runner / greeting-responder 例子）
```

### agent 列表是 inline 还是 attachment

`shouldInjectAgentListInMessages()`（`prompt.ts:59-64`）：

```typescript
function shouldInjectAgentListInMessages(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES)) return true
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES)) return false
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false)
}
```

注释解释（`prompt.ts:48-56`）：

> The dynamic agent list was ~10.2% of fleet cache_creation tokens: MCP async connect, /reload-plugins, or permission-mode changes mutate the list → description changes → full tool-schema cache bust.

也就是说：把 agent 列表写在工具 description 里 = 每次列表变化都会击穿 tool schema cache。后来改成把 agent 列表放到一条独立 attachment message 里：

```text
description（被 cache）：
  "Available agent types are listed in <system-reminder> messages in the conversation."

attachment（每轮可能变）：
  "<agent_listing_delta>...</agent_listing_delta>"
```

代价：模型每轮要去 system-reminder 里查 agent 列表；好处：tool description 稳定，不击穿 cache。

## checkPermissions

```typescript
async checkPermissions(input, context): Promise<PermissionResult> {
  const appState = context.getAppState()

  // 仅 auto mode (ant) 路由到分类器
  if ("external" === 'ant' && appState.toolPermissionContext.mode === 'auto') {
    return {
      behavior: 'passthrough',
      message: 'Agent tool requires permission to spawn sub-agents.'
    }
  }
  return { behavior: 'allow', updatedInput: input }
}
```

- **auto 模式（ant 内部）**：`passthrough` 给 auto-classifier，由分类器决定能否 spawn。
- **其它所有模式**：直接 `allow`，不弹 UI。

这是 `isReadOnly() === true` 的含义体现——`AgentTool` 自己不要权限，权限管控下放到子代理的工具调用。

## call() 的整体结构

接 02 篇的决策树展开，`AgentTool.call()` 的源码骨架（`AgentTool.tsx:239-1262`）大致如下：

```text
async call({ prompt, subagent_type, description, model, run_in_background, name, team_name, mode, isolation, cwd }, toolUseContext, canUseTool, assistantMessage, onProgress?) {
  const startTime = Date.now()
  const model = isCoordinatorMode() ? undefined : modelParam
  const appState = toolUseContext.getAppState()
  const permissionMode = appState.toolPermissionContext.mode
  const rootSetAppState = toolUseContext.setAppStateForTasks ?? toolUseContext.setAppState

  // [1] 入口守卫
  if (team_name && !isAgentSwarmsEnabled()) throw "Agent Teams not available"
  const teamName = resolveTeamName({ team_name }, appState)
  if (isTeammate() && teamName && name) throw "Teammates cannot spawn other teammates"
  if (isInProcessTeammate() && teamName && run_in_background === true) throw "..."

  // [2] swarm 派发
  if (teamName && name) {
    const result = await spawnTeammate({...}, toolUseContext)
    return { data: { status: 'teammate_spawned', prompt, ...result.data } }
  }

  // [3] 解析 selectedAgent（fork 路径 / 显式 type / 默认 general-purpose）
  const effectiveType = subagent_type ?? (isForkSubagentEnabled() ? undefined : GENERAL_PURPOSE_AGENT.agentType)
  const isForkPath = effectiveType === undefined
  let selectedAgent: AgentDefinition
  if (isForkPath) {
    if (toolUseContext.options.querySource === 'agent:builtin:fork' || isInForkChild(...))
      throw "Fork is not available inside a forked worker."
    selectedAgent = FORK_AGENT
  } else {
    // 查 agent 是否被 permission deny，给精确错误
    ...
    selectedAgent = found
  }

  // [4] background 字段检查 + requiredMcpServers 等待 + 颜色初始化
  if (isInProcessTeammate() && teamName && selectedAgent.background === true) throw "..."
  if (requiredMcpServers?.length) {
    // 等待 pending MCP server，最多 30s
    ...
    if (!hasRequiredMcpServers(...)) throw "Agent ... requires MCP servers matching: ..."
  }
  if (selectedAgent.color) setAgentColor(selectedAgent.agentType, selectedAgent.color)

  // [5] 模型解析 + analytics
  const resolvedAgentModel = getAgentModel(selectedAgent.model, parentModel, isForkPath ? undefined : model, permissionMode)
  logEvent('tengu_agent_tool_selected', {...})

  // [6] isolation: remote (ant only)
  const effectiveIsolation = isolation ?? selectedAgent.isolation
  if ("external" === 'ant' && effectiveIsolation === 'remote') {
    const eligibility = await checkRemoteAgentEligibility()
    if (!eligibility.eligible) throw "..."
    const session = await teleportToRemote({...})
    const { taskId, sessionId } = registerRemoteAgentTask({...})
    return { data: { status: 'remote_launched', ... } }
  }

  // [7] 构造 system prompt + prompt messages
  if (isForkPath) {
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt ?? recompute()
    promptMessages = buildForkedMessages(prompt, assistantMessage)
  } else {
    const agentPrompt = selectedAgent.getSystemPrompt({ toolUseContext })
    enhancedSystemPrompt = await enhanceSystemPromptWithEnvDetails([agentPrompt], resolvedAgentModel, additionalWorkingDirectories)
    promptMessages = [createUserMessage({ content: prompt })]
  }

  // [8] shouldRunAsync 决策
  const forceAsync = isForkSubagentEnabled()
  const assistantForceAsync = feature('KAIROS') ? appState.kairosEnabled : false
  const shouldRunAsync = (run_in_background === true || selectedAgent.background === true || isCoordinator || forceAsync || assistantForceAsync || (proactiveModule?.isProactiveActive() ?? false)) && !isBackgroundTasksDisabled

  // [9] 构造 worker tool pool（独立于父的 permission mode）
  const workerPermissionContext = { ...appState.toolPermissionContext, mode: selectedAgent.permissionMode ?? 'acceptEdits' }
  const workerTools = assembleToolPool(workerPermissionContext, appState.mcp.tools)

  // [10] isolation: worktree
  const earlyAgentId = createAgentId()
  let worktreeInfo: WorktreeInfo | null = null
  if (effectiveIsolation === 'worktree') {
    worktreeInfo = await createAgentWorktree(`agent-${earlyAgentId.slice(0, 8)}`)
  }
  if (isForkPath && worktreeInfo) {
    promptMessages.push(createUserMessage({ content: buildWorktreeNotice(getCwd(), worktreeInfo.worktreePath) }))
  }

  // [11] runAgentParams 拼装
  const runAgentParams = { agentDefinition, promptMessages, toolUseContext, canUseTool, isAsync: shouldRunAsync, querySource, model, override, availableTools, forkContextMessages, useExactTools, worktreePath, description }
  const wrapWithCwd = <T,>(fn) => cwdOverridePath ? runWithCwdOverride(cwdOverridePath, fn) : fn()

  // [12] 派发 async / sync
  if (shouldRunAsync) {
    const asyncAgentId = earlyAgentId
    const agentBackgroundTask = registerAsyncAgent({...})
    if (name) rootSetAppState(prev => { ... agentNameRegistry.set(name, asyncAgentId) })
    void runWithAgentContext(asyncAgentContext, () => wrapWithCwd(() => runAsyncAgentLifecycle({...})))
    return { data: { status: 'async_launched', agentId, outputFile, ... } }
  }

  // [13] 同步：迭代 runAgent generator，收集 messages，finalize
  // ... 大块 try/catch/finally 处理 auto-background timer / foreground task / cleanup
  const agentResult = finalizeAgentTool(agentMessages, syncAgentId, metadata)
  return { data: { status: 'completed', prompt, ...agentResult, ...worktreeResult } }
}
```

## mapToolResultToToolResultBlockParam 的 5 个分支

按 status 字段映射回 `tool_result.content`（`AgentTool.tsx:1298-1380`）：

### teammate_spawned

```text
Spawned successfully.
agent_id: <teammate_id>
name: <name>
team_name: <team_name>
The agent is now running and will receive instructions via mailbox.
```

### remote_launched

```text
Remote agent launched in CCR.
taskId: <id>
session_url: <url>
output_file: <path>
The agent is running remotely. You will be notified automatically when it completes.
Briefly tell the user what you launched and end your response.
```

### async_launched

```text
Async agent launched successfully.
agentId: <id> (internal ID - do not mention to user. Use SendMessage with to: '<id>' to continue this agent.)
The agent is working in the background. You will be notified automatically when it completes.

[canReadOutputFile = true:]
Do not duplicate this agent's work — avoid working with the same files or topics it is using. Work on non-overlapping tasks, or briefly tell the user what you launched and end your response.
output_file: <path>
If asked, you can check progress before completion by using FileRead or Bash tail on the output file.

[canReadOutputFile = false:]
Briefly tell the user what you launched and end your response. Do not generate any other text — agent results will arrive in a subsequent message.
```

### completed（同步成功）

```text
[子代理最后一条 assistant message 的 content blocks]
[+ 可选 worktreePath / worktreeBranch 尾巴]
[+ 可选 handoff warning（TRANSCRIPT_CLASSIFIER 启用时）]
```

或者 `(Subagent completed but returned no output.)`（子代理跑完没产出内容时的 placeholder）。

### completed 的"无产出" placeholder

```typescript
const contentOrMarker = data.content.length > 0 ? data.content : [{
  type: 'text' as const,
  text: '(Subagent completed but returned no output.)'
}]
```

注释解释：子代理跑完没文本时，tool_result 如果只有 metadata trailer（agentId / usage），某些模型会读成"没事可做"直接结束 turn。所以显式塞一句让父代理"有东西可反应"。

## auto-background 机制

`AgentTool.tsx:72-77`：

```typescript
function getAutoBackgroundMs(): number {
  if (isEnvTruthy(process.env.CLAUDE_AUTO_BACKGROUND_TASKS) || getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents', false)) {
    return 120_000  // 2 分钟
  }
  return 0  // 关闭
}
```

启用时：同步 spawn 的子代理跑超过 120 秒会自动转到后台。父代理收到 `async_launched` 风格的 `tool_result`，原 turn 继续。源码里这个转换叫 "backgrounding"，对应 `wasBackgrounded` 标志位。`AgentTool.tsx:950-1040` 的 backgrounded 分支处理这种"中途转后台"的清理：

- foreground summarization 停掉
- 切到 backgrounded summarization
- 注册到 LocalAgentTask 走 async lifecycle
- worktree cleanup 推迟到 backgrounded 路径

这是给"我不知道这个 agent 会跑多久"的兜底——既不强制后台，又不让父对话卡死。

## UI 串流

`onProgress` 回调（`call()` 签名的最后一个参数）把子代理的 normalize 后 messages 回灌父对话 UI：

```typescript
if (onProgress) {
  onProgress({
    toolUseID: `agent_${assistantMessage.message.id}`,
    data: {
      message: m,
      type: 'agent_progress',
      prompt: '',
      agentId: syncAgentId
    }
  })
}
```

`UI.tsx:renderToolUseProgressMessage` 拿到 `agent_progress` 类型的 progress，渲染成"嵌套在父 tool use 下的 grouped 子 tool use 流"。用户看到的是树形展开的进度。

源码里专门有一个 `renderGroupedAgentToolUse`（`UI.tsx`），把同一个 agent spawn 的所有内部 tool use 折叠成一个分组渲染。

## SyntheticOutput 与 abort 处理

`AgentTool.tsx:1207-1218`：

```typescript
const lastMessage = agentMessages.findLast(_ => _.type !== 'system' && _.type !== 'progress')
if (lastMessage && isSyntheticMessage(lastMessage)) {
  logEvent('tengu_agent_tool_terminated', {...})
  throw new AbortError()
}
```

`SyntheticMessage` 表示"runtime 注入的合成 message"，比如 `REJECT_MESSAGE` 或 stream fallback 的 placeholder。子代理最后留下一条合成 message 通常意味着 abort 中断——这时不应返回 partial result，而是显式 throw `AbortError`，让 query 主循环知道这一轮被打断。

## 几个容易踩的点

1. **`description` 不是 prompt 的简化**：它是 3-5 词的 UI 标签，单独存在。模型若把任务塞到 description 里，UI 显示会很丑（一行字过长）。
2. **`run_in_background: false` 不一定能让 agent 同步跑**：`selectedAgent.background === true` 或 fork 启用、KAIROS 等都会强制 async。
3. **`isolation: "worktree"` 不等于"独立进程"**：worktree 只切 cwd 和文件视图，进程仍是同一个；async/teammate/remote 才涉及独立进程或独立 task lifecycle。
4. **同步 spawn 也可能"突然变后台"**：auto-background 在 120s 后强制转后台，模型读到的 tool_result 会从 `completed` 变成 `async_launched`——这个分支模型必须能处理两种形态。
5. **agent 列表在 prompt 还是 attachment**：默认 prompt（GB 关）；GB 开启时切到 attachment。两种模式下父代理读到的"可用 agent 列表"位置不同，调试时容易找错地方。
6. **`team_name` 单独传不进 swarm**：必须 `team_name + name` 配对才进入 `teammate_spawned` 分支。单独 `team_name` 时 `resolveTeamName` 会返回 undefined（视为不指定 team），走默认派发。
