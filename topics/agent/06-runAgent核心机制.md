# 06 - runAgent 核心机制

## 文件定位

`src/tools/AgentTool/runAgent.ts`（973 LOC）是子代理的运行时核心。`AgentTool.call()`（05 篇）把"派发什么 agent、用什么模式、跑在哪个 cwd"决策好后，把所有参数打包传给 `runAgent`。`runAgent` 是个 `AsyncGenerator<Message, void>`——同步 spawn 时父代理直接 `for await` 迭代；async spawn 时被包在 `runAsyncAgentLifecycle` 里 void 调用。

## 函数签名

`runAgent.ts:248-329`：

```typescript
export async function* runAgent({
  agentDefinition,              // 选定的 agent 定义（含 systemPrompt / tools / model 等）
  promptMessages,               // 首轮 messages（fork 情况下包含父历史 + 指令）
  toolUseContext,               // 父代理传来的运行时上下文
  canUseTool,                   // 权限检查回调
  isAsync,                      // 是否后台运行
  canShowPermissionPrompts,     // 是否能弹权限 UI（默认 !isAsync）
  forkContextMessages,          // fork 模式下父对话的 messages（先于 promptMessages）
  querySource,                  // analytics 用，agent:builtin:explore 这种字符串
  override,                     // { userContext?, systemContext?, systemPrompt?, abortController?, agentId? }
  model,                        // 工具入参的 model 覆盖（family alias）
  maxTurns,                     // 内部循环最大轮数（agent 定义可设）
  preserveToolUseResults,       // in-process teammate 的 transcript 保留
  availableTools,               // 预组装的 worker tool pool（AgentTool 算好的）
  allowedTools,                 // session allow rules（替换父的）
  onCacheSafeParams,            // 后台 summarization 用的回调
  contentReplacementState,      // resume 时的 replacement state
  useExactTools,                // fork 路径：旁路 tool filter
  worktreePath,                 // worktree 路径，写入 metadata 供 resume
  description,                  // 原始任务描述
  transcriptSubdir,             // transcript 分组子目录
  onQueryProgress,              // 长流活跃度回调
}): AsyncGenerator<Message, void>
```

## 共享、克隆、隔离三类状态

子代理与父代理的关系不是"完全独立"，也不是"全部继承"。`runAgent` 按字段决定每一项是 共享 / 克隆 / 隔离 中的哪一种。

| 字段 | 关系 | 说明 |
|------|------|------|
| `getAppState` / AppState 主体 | 共享 | 通过 `toolUseContext.getAppState` 读，父代理对 AppState 的改动子代理能看到 |
| `setAppState` | 共享（同步）/ 隔离（async） | sync 子代理直接调父的 setAppState；async 子代理用 `rootSetAppState`（绕过 in-process teammate no-op）|
| `readFileState`（file read 缓存） | 克隆 | fork 时 `cloneFileStateCache(parent.readFileState)`；非 fork 重建空的 |
| `abortController` | 共享（sync）/ 独立（async） | sync 用父的，ESC 一起死；async 新建独立 controller |
| `mcpClients` | 共享 + 增量 | 继承父的 mcpClients；agent.mcpServers 中声明的额外 server 增加进来 |
| `agentId` | 隔离 | `createAgentId()` 每次新建；override.agentId 优先 |
| `transcript` | 隔离（subdir） | sidechain transcript 落到 `subagents/<agentId>/` |
| `mainLoopModel` | 替换 | `resolvedAgentModel`（04 篇）覆盖父的 |
| `thinkingConfig` | 替换（默认）/ 继承（fork） | 普通子代理 `{ type: 'disabled' }`；fork 继承父 |
| `isNonInteractiveSession` | 替换（默认 true if async）/ 继承（fork） | async 子代理强制 non-interactive |
| `userContext`（含 CLAUDE.md） | 重读 / 删 ClaudeMd（omitClaudeMd 时）| 不复制父的 userContext 引用 |
| `systemContext`（含 gitStatus） | 重读 / 删 gitStatus（Explore/Plan）| 同上 |
| `commands` | 替换为 `[]` | 子代理拿不到 slash commands |
| `permissionMode` | agent 定义覆盖（条件） | 除非父是 bypassPermissions / acceptEdits（永远优先）|
| `effortValue` | agent 定义覆盖 | agent.effort 设了就替换，否则继承 |
| `allowedAgentTypes` | 父的 `options.agentDefinitions.allowedAgentTypes` 传递 | 子代理也能 spawn 但受同样约束 |
| `agentNameRegistry` | 共享 | 子代理 spawn 的 async agent 也写到根注册表 |
| `hooks` | 增量注册 | agent.hooks 在 spawn 时注册，结束时 clearSessionHooks |
| `todos[agentId]` | 隔离 | 各 agentId 独立空间，结束时清理 |

## fork 与非 fork 的双路径

`runAgent` 的内部逻辑有两条主路径，由 `useExactTools` 切换：

```typescript
// runAgent.ts:500-502
const resolvedTools = useExactTools
  ? availableTools                                          // fork: 用父的完整工具数组
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools  // 普通：过滤
```

```typescript
// runAgent.ts:667-695
const agentOptions: ToolUseContext['options'] = {
  isNonInteractiveSession: useExactTools
    ? toolUseContext.options.isNonInteractiveSession      // fork: 继承
    : isAsync ? true : (parent ?? false),                  // 普通: async 强制 true
  ...
  thinkingConfig: useExactTools
    ? toolUseContext.options.thinkingConfig               // fork: 继承
    : { type: 'disabled' as const },                       // 普通: 禁用
  ...
  ...(useExactTools && { querySource })                   // fork: 写入 querySource 供递归 fork 守卫
}
```

为什么 fork 要继承这些？因为 fork 的目的是 prompt cache 共享，**子代理的 API 请求前缀必须和父对话完全一致**。任何不一致（tools 数组不同、thinking 配置不同）都会破 cache。

## 内部 query 循环

`runAgent.ts:747-806`（精简后）：

```typescript
let lastRecordedUuid: UUID | null = initialMessages.at(-1)?.uuid ?? null

try {
  for await (const message of query({
    messages: initialMessages,
    systemPrompt: agentSystemPrompt,
    userContext: resolvedUserContext,
    systemContext: resolvedSystemContext,
    canUseTool,
    toolUseContext: agentToolUseContext,
    querySource,
    maxTurns: maxTurns ?? agentDefinition.maxTurns,
  })) {
    onQueryProgress?.()

    // stream_event 不 yield 但喂 TTFT metrics
    if (message.type === 'stream_event' && message.event.type === 'message_start' && message.ttftMs != null) {
      toolUseContext.pushApiMetricsEntry?.(message.ttftMs)
      continue
    }

    // attachment 类（如 max_turns_reached）特殊处理
    if (message.type === 'attachment') {
      if (message.attachment.type === 'max_turns_reached') {
        logForDebugging(`[Agent: ...] Reached max turns limit (...)`)
        break
      }
      yield message
      continue
    }

    if (isRecordableMessage(message)) {
      await recordSidechainTranscript([message], agentId, lastRecordedUuid).catch(...)
      if (message.type !== 'progress') lastRecordedUuid = message.uuid
      yield message
    }
  }

  if (agentAbortController.signal.aborted) throw new AbortError()

  // built-in 的 callback 钩子
  if (isBuiltInAgent(agentDefinition) && agentDefinition.callback) {
    agentDefinition.callback()
  }
} finally {
  // === 9 步清理（见 09 篇详细）===
  await mcpCleanup()
  if (agentDefinition.hooks) clearSessionHooks(rootSetAppState, agentId)
  if (feature('PROMPT_CACHE_BREAK_DETECTION')) cleanupAgentTracking(agentId)
  agentToolUseContext.readFileState.clear()
  initialMessages.length = 0
  unregisterPerfettoAgent(agentId)
  clearAgentTranscriptSubdir(agentId)
  rootSetAppState(prev => {
    if (!(agentId in prev.todos)) return prev
    const { [agentId]: _removed, ...todos } = prev.todos
    return { ...prev, todos }
  })
  killShellTasksForAgent(agentId, toolUseContext.getAppState, rootSetAppState)
  if (feature('MONITOR_TOOL')) killMonitorMcpTasksForAgent(...)
}
```

关键点：

- 子代理跑的是和主对话**完全一样的** `query()` 主循环（同一个 query engine），只是传入的 context 不同。**子代理不是"另一种东西"，就是另一份 query 实例。**
- 每条 recordable message 都通过 `recordSidechainTranscript` 增量落盘（O(1) per message），不会等到 agent 结束才批量写。
- max_turns_reached 触发 `break`，不抛错。
- abort 后才抛 `AbortError`，让外层知道是用户取消。

## 系统提示的两条构造路径

**fork 路径**（cache-identical）：

```typescript
// AgentTool.tsx:496-511
forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
  ?? buildEffectiveSystemPrompt({...})  // 回退：重新计算（注释说可能 cache 飘）
```

直接用父对话渲染好的系统提示字节。

**非 fork 路径**（重新构造）：

```typescript
// runAgent.ts:506-518
const agentSystemPrompt = override?.systemPrompt
  ? override.systemPrompt
  : asSystemPrompt(await getAgentSystemPrompt(
      agentDefinition,
      toolUseContext,
      resolvedAgentModel,
      additionalWorkingDirectories,
      resolvedTools,  // 关键：传 enabled tools 给 prompt enhancer
    ))

// runAgent.ts:906-932
async function getAgentSystemPrompt(...): Promise<string[]> {
  const enabledToolNames = new Set(resolvedTools.map(t => t.name))
  try {
    const agentPrompt = agentDefinition.getSystemPrompt({ toolUseContext })
    const prompts = [agentPrompt]
    return await enhanceSystemPromptWithEnvDetails(
      prompts,
      resolvedAgentModel,
      additionalWorkingDirectories,
      enabledToolNames,
    )
  } catch (_error) {
    return enhanceSystemPromptWithEnvDetails(
      [DEFAULT_AGENT_PROMPT],
      resolvedAgentModel,
      additionalWorkingDirectories,
      enabledToolNames,
    )
  }
}
```

`enhanceSystemPromptWithEnvDetails` 是把 cwd / OS / git status / 当前可用工具列表 / shell 等环境信息追加到系统提示。

## userContext 与 systemContext 的裁剪

```typescript
// runAgent.ts:380-410
const [baseUserContext, baseSystemContext] = await Promise.all([
  override?.userContext ?? getUserContext(),     // CLAUDE.md 层级
  override?.systemContext ?? getSystemContext(), // gitStatus 等
])

// omitClaudeMd 优化
const shouldOmitClaudeMd =
  agentDefinition.omitClaudeMd &&
  !override?.userContext &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_slim_subagent_claudemd', true)
const { claudeMd: _omittedClaudeMd, ...userContextNoClaudeMd } = baseUserContext
const resolvedUserContext = shouldOmitClaudeMd ? userContextNoClaudeMd : baseUserContext

// Explore/Plan 专属：去掉 gitStatus（最多 40KB，标记为 stale）
const { gitStatus: _omittedGitStatus, ...systemContextNoGit } = baseSystemContext
const resolvedSystemContext = (agentDefinition.agentType === 'Explore' || agentDefinition.agentType === 'Plan')
  ? systemContextNoGit
  : baseSystemContext
```

注释解释（`runAgent.ts:385-410`）：

- omitClaudeMd 默认在 `tengu_slim_subagent_claudemd` GB 启用时生效，read-only agent（Explore / Plan）跳过 CLAUDE.md。注释里写 "Saves ~5-15 Gtok/week across 34M+ Explore spawns" —— 因为 Explore spawn 频率极高，每次省几千 token 累计就是数 GTok。
- gitStatus 对 Explore/Plan 也是死重量——它们要查 git 自己跑 `git status`，session 启动时的快照通常已经 stale。

## permission mode 与 allowedTools

`agentGetAppState`（`runAgent.ts:416-498`）每次返回**动态合成**的 AppState，按 agent 定义覆盖 permission 字段：

```typescript
const agentGetAppState = () => {
  const state = toolUseContext.getAppState()
  let toolPermissionContext = state.toolPermissionContext

  // 覆盖 permission mode（但 bypassPermissions/acceptEdits 始终优先）
  if (
    agentPermissionMode &&
    state.toolPermissionContext.mode !== 'bypassPermissions' &&
    state.toolPermissionContext.mode !== 'acceptEdits' &&
    !(feature('TRANSCRIPT_CLASSIFIER') && state.toolPermissionContext.mode === 'auto')
  ) {
    toolPermissionContext = { ...toolPermissionContext, mode: agentPermissionMode }
  }

  // async agent 不能弹 UI，禁权限 prompt
  const shouldAvoidPrompts =
    canShowPermissionPrompts !== undefined
      ? !canShowPermissionPrompts
      : agentPermissionMode === 'bubble' ? false : isAsync
  if (shouldAvoidPrompts) {
    toolPermissionContext = { ...toolPermissionContext, shouldAvoidPermissionPrompts: true }
  }

  // 后台 agent 即使能弹 prompt 也先等自动检查
  if (isAsync && !shouldAvoidPrompts) {
    toolPermissionContext = { ...toolPermissionContext, awaitAutomatedChecksBeforeDialog: true }
  }

  // allowedTools：替换 session rule，保留 cliArg rule
  if (allowedTools !== undefined) {
    toolPermissionContext = {
      ...toolPermissionContext,
      alwaysAllowRules: {
        cliArg: state.toolPermissionContext.alwaysAllowRules.cliArg,
        session: [...allowedTools],
      },
    }
  }

  // effort 覆盖
  const effortValue = agentDefinition.effort !== undefined ? agentDefinition.effort : state.effortValue

  if (toolPermissionContext === state.toolPermissionContext && effortValue === state.effortValue) {
    return state
  }
  return { ...state, toolPermissionContext, effortValue }
}
```

要点：

- 父代理在 `bypassPermissions` / `acceptEdits` 时，子代理**没法降权限**。这是反向逃逸保护——父信任度高时子代理跟随。
- async / bubble 模式的 prompt 行为差异：bubble 模式专门用来把 prompt 抛回父对话终端，所以即使是 async（无主线 UI）也可能产生 UI 操作。
- `allowedTools` 是 SDK / API 用户从外部传入的"会话级允许列表"，会替换父对话级别的 session allow rules，保留 CLI flag 级别的（防止 SDK 用户的 --allowedTools 被吃掉）。

## SubagentStart hooks

`runAgent.ts:530-555`：

```typescript
const additionalContexts: string[] = []
for await (const hookResult of executeSubagentStartHooks(
  agentId,
  agentDefinition.agentType,
  agentAbortController.signal,
)) {
  if (hookResult.additionalContexts && hookResult.additionalContexts.length > 0) {
    additionalContexts.push(...hookResult.additionalContexts)
  }
}

if (additionalContexts.length > 0) {
  const contextMessage = createAttachmentMessage({
    type: 'hook_additional_context',
    content: additionalContexts,
    hookName: 'SubagentStart',
    toolUseID: randomUUID(),
    hookEvent: 'SubagentStart',
  })
  initialMessages.push(contextMessage)
}
```

SubagentStart hook 允许"在子代理首轮 message 之前注入额外上下文"——例如自动喂入项目级 README、用户偏好等。返回的 `additionalContexts[]` 被打包成一条 attachment user message，放在 initialMessages 末尾（即子代理首轮 user prompt 之前最新插入）。

## frontmatter hooks 与 skills 预加载

```typescript
// runAgent.ts:557-575
const hooksAllowedForThisAgent = !isRestrictedToPluginOnly('hooks') || isSourceAdminTrusted(agentDefinition.source)
if (agentDefinition.hooks && hooksAllowedForThisAgent) {
  registerFrontmatterHooks(rootSetAppState, agentId, agentDefinition.hooks, `agent '${agentDefinition.agentType}'`, true)
}

// runAgent.ts:577-646 - skills 预加载（按 frontmatter 的 skills 数组）
const skillsToPreload = agentDefinition.skills ?? []
if (skillsToPreload.length > 0) {
  const allSkills = await getSkillToolCommands(getProjectRoot())
  const validSkills = [...]  // 经过 resolveSkillName 校验
  const loaded = await Promise.all(validSkills.map(async ({skillName, skill}) => ({
    ...
    content: await skill.getPromptForCommand('', toolUseContext)
  })))
  for (const { skillName, skill, content } of loaded) {
    initialMessages.push(createUserMessage({
      content: [{ type: 'text', text: formatSkillLoadingMetadata(skillName, skill.progressMessage) }, ...content],
      isMeta: true,
    }))
  }
}
```

预加载的 skills 表现为隐藏 meta user message（`isMeta: true`），不在 UI 显示但会进入 API 请求。`registerFrontmatterHooks` 把 agent 自带的 hooks 注册到 session（agent 结束时清理）。`isAgent: true` 让 `Stop` hook 转换成 `SubagentStop`——子代理结束 ≠ session 结束。

## agent-specific MCP servers

```typescript
// runAgent.ts:648-664
const { clients: mergedMcpClients, tools: agentMcpTools, cleanup: mcpCleanup } =
  await initializeAgentMcpServers(agentDefinition, toolUseContext.options.mcpClients)

const allTools = agentMcpTools.length > 0
  ? uniqBy([...resolvedTools, ...agentMcpTools], 'name')
  : resolvedTools
```

agent 定义里的 `mcpServers` 是 additive 的：在父 MCP 基础上**附加**这个 agent 专用的 server，子代理 spawn 时启动它们的 client，agent 结束时 cleanup。这让一个 agent 可以声明"我需要 Playwright"，不污染主对话的 MCP 池。

## metadata 持久化

```typescript
// runAgent.ts:735-742
void recordSidechainTranscript(initialMessages, agentId).catch(...)
void writeAgentMetadata(agentId, {
  agentType: agentDefinition.agentType,
  ...(worktreePath && { worktreePath }),
  ...(description && { description }),
}).catch(...)
```

写两份持久化：

1. **sidechain transcript**：完整 message 流，落到 `subagents/<agentId>/transcript.jsonl`。
2. **agent metadata**：JSON 文件，存 `agentType` / `worktreePath` / `description`。供 `resumeAgentBackground`（09 篇）从磁盘恢复一个之前后台跑过的 agent。

两份都 fire-and-forget——持久化失败不阻塞 agent 跑。

## `agentToolUseContext` 与 `createSubagentContext`

```typescript
// runAgent.ts:700-714
const agentToolUseContext = createSubagentContext(toolUseContext, {
  options: agentOptions,
  agentId,
  agentType: agentDefinition.agentType,
  messages: initialMessages,
  readFileState: agentReadFileState,
  abortController: agentAbortController,
  getAppState: agentGetAppState,
  shareSetAppState: !isAsync,           // sync 共享父 setAppState，async 不共享
  shareSetResponseLength: true,
  criticalSystemReminder_EXPERIMENTAL: agentDefinition.criticalSystemReminder_EXPERIMENTAL,
  contentReplacementState,
})

if (preserveToolUseResults) {
  agentToolUseContext.preserveToolUseResults = true
}
```

`createSubagentContext` 是个 utility（在 query 模块里），按字段决定哪些回调直接复用父的引用、哪些重建。`shareSetAppState: !isAsync` 是核心分流：

- **sync 子代理**：直接调父的 `setAppState`——session-scoped 改动会反映到主进程 AppState。
- **async 子代理**：不共享 `setAppState`（设置改动只在 async 子代理自己的 context 里）；但需要写主进程的 task progress / kill state 时走 `setAppStateForTasks`（即 `rootSetAppState`），它绕过 in-process teammate 的 no-op。

## 最大 turn 限制

```typescript
maxTurns: maxTurns ?? agentDefinition.maxTurns
```

`runAgent` 的入参 `maxTurns` 优先，否则用 agent 定义的 `maxTurns`，再否则 query 默认（在 `query.ts` 里有上限）。`FORK_AGENT.maxTurns = 200`（`forkSubagent.ts:65`）。达到上限后 query 通过 `attachment.type === 'max_turns_reached'` 通知 runAgent，runAgent `break` 出循环。

## 后台 summarization 钩子

```typescript
// runAgent.ts:721-730
if (onCacheSafeParams) {
  onCacheSafeParams({
    systemPrompt: agentSystemPrompt,
    userContext: resolvedUserContext,
    systemContext: resolvedSystemContext,
    toolUseContext: agentToolUseContext,
    forkContextMessages: initialMessages,
  })
}
```

`CacheSafeParams` 包了一份"prompt cache 一致"的快照。后台 summarization 会**复用同样的前缀**起一个 side query，定期 fork 子代理的当前对话做总结，喂回父对话的 progress 显示。共享前缀保证 summarization 的 cache hit，几乎免费。这个机制启用条件在 `agentToolUtils.ts:runAsyncAgentLifecycle` 的 `enableSummarization` 字段。

## 三个不同的 abortController 来源

```typescript
// runAgent.ts:524-528
const agentAbortController =
  override?.abortController          // 1. AgentTool.tsx 显式传入（async 路径用 task.abortController）
  ?? (isAsync
      ? new AbortController()        // 2. async 无 override：新建独立的
      : toolUseContext.abortController)  // 3. sync 无 override：共享父的
```

- async 路径（`registerAsyncAgent` 后）：`override.abortController = agentBackgroundTask.abortController`，由 LocalAgentTask 维护，`chat:killAgents` 触发它。
- sync 路径：共享父的 controller，ESC 杀父 = 杀子。
- 极少数情况（手动测试、内部 API）：override 为空且 sync —— 仍走父 controller。

## 一图概览 runAgent 的状态拓扑

```text
父代理 toolUseContext
   ├── getAppState ────────── 子: 共享（agentGetAppState 包装一层覆盖 permission）
   ├── setAppState ────────── 子: sync 共享 / async 不共享
   ├── setAppStateForTasks ── 子: 共享（task 维度的根 store）
   ├── abortController ────── 子: sync 共享 / async 独立
   ├── readFileState ─────── 子: fork 克隆 / 非 fork 重建
   ├── mcpClients ─────────── 子: 共享 + agent.mcpServers 增量
   ├── options.tools ──────── 子: 用 availableTools（外部预算）/ fork 时 useExactTools 直传
   ├── options.commands ───── 子: 替换为 []
   ├── options.mainLoopModel ─ 子: 替换为 resolvedAgentModel
   ├── options.thinkingConfig 子: 替换为 disabled / fork 时继承
   ├── messages ───────────── 子: forkContextMessages + promptMessages + SubagentStart hooks + 预加载 skills
   ├── agentId ────────────── 子: createAgentId() / override
   ├── transcript ─────────── 子: subagents/<agentId>/ 独立 sidechain
   ├── hooks ──────────────── 子: 注册 agent.hooks，结束时清理
   └── userContext/systemContext 子: 重读 + 可能裁剪（omitClaudeMd / Explore-Plan gitStatus）

子代理 agentToolUseContext
   ↓
query() 主循环（同样的 query engine）
   ↓
StreamingToolExecutor 调度内部工具（与父独立的实例）
   ↓
yield Message → 父代理 for await 接收
```

## 容易踩的点

1. **"子代理是独立进程"是错的**：在同一个 Node 进程，复用 query engine，只是 context 隔开。teammate 和 remote 才涉及独立进程。
2. **omitClaudeMd 不是 hard cut**：还要 `tengu_slim_subagent_claudemd` GB（默认 true）开启。GB 关闭时 omitClaudeMd 失效。
3. **`override?.userContext` 显式传入时跳过 omitClaudeMd**：注释专门说"Explicit override.userContext from callers is preserved untouched"。如果你手动塞了 userContext，agent 设的 omitClaudeMd 不会偷偷裁剪。
4. **`isAsync: true` 不等于"在后台 process"**：仍然是同进程的 Promise，只是不阻塞父 turn。process-level 隔离是 teammate（tmux process）和 remote（CCR）的事。
5. **子代理可以再 spawn 子代理（孙代理）**：只要 `AgentTool` 在 `availableTools` 里。built-in Explore / Plan / verification 在定义里禁了 `AGENT_TOOL_NAME`，所以它们不能套娃。fork 子代理保留 Agent 工具但通过 querySource 守卫禁止再 fork。
6. **`maxTurns` 命中后是 break 不是 throw**：所以 max_turns_reached 时子代理"正常结束"，父代理收到的 result 是当时收集到的最后内容——可能 truncated。如果模型 prompt 没要求"先 summary 再用完 turn"，结果就会是断尾。
