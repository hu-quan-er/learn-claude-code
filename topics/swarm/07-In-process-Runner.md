# 07 In-process Runner

> `inProcessRunner.ts` **1552 行**——为什么这么长？因为它要在同一进程里"装一份完整的 Claude"。本篇拆它的状态机：runWithTeammateContext 进入隔离作用域、claim task → run agent → wait for next prompt / shutdown 的主循环、per-turn abort、task-list 自动认领、idle 通知。

## 7.1 inProcessRunner 在系统里的位置

回顾 [04 In-process backend](./04-In-process-backend.md) 的 spawn 流程：

```
InProcessBackend.spawn()
  → spawnInProcessTeammate (创建 context + 注册任务)
  → startInProcessTeammate(config)  ← 本篇主角
        │
        ▼
   inProcessRunner.ts:1544
   startInProcessTeammate (fire-and-forget)
        │
        ▼
   1. runWithTeammateContext (AsyncLocalStorage 包装)
        │
        ▼
   2. runInProcessTeammate (实际执行)
        │
        ▼
        └─ 内部 while loop 跑 teammate 完整生命周期
```

InProcessBackend 调 `startInProcessTeammate` fire-and-forget 启动 → runner 接管，跑 teammate 的完整生命周期直到 shutdown_approved 或 abort。

## 7.2 InProcessRunnerConfig —— 16 个字段的复杂度来源

`inProcessRunner.ts:471-502`：

```ts
export type InProcessRunnerConfig = {
  identity: TeammateIdentity                // agentId/name/teamName/color/planModeRequired
  taskId: string                             // AppState.tasks 里的 task id
  prompt: string                             // 初始 prompt
  agentDefinition?: CustomAgentDefinition    // 自定义 agent 类型
  teammateContext: TeammateContext           // AsyncLocalStorage 用的 context
  toolUseContext: ToolUseContext             // parent 的 tool 调用上下文
  abortController: AbortController           // 生命周期 abort (独立, 不绑 parent)
  model?: string                             // 模型覆盖
  systemPrompt?: string                      // system prompt 覆盖
  systemPromptMode?: 'default' | 'replace' | 'append'  // system prompt 模式
  allowedTools?: string[]                    // 允许的工具列表
  allowPermissionPrompts?: boolean           // 允不允许弹权限对话框
  description?: string                       // 初始 prompt 摘要
  invokingRequestId?: string                 // analytics lineage
}
```

16 个字段——每个对应一个独立的可配置点：

- **身份相关** (5)：identity 内含 agentId/name/teamName/color/planModeRequired
- **任务相关** (3)：taskId、prompt、description
- **agent 定义** (1)：agentDefinition（custom agent 类型）
- **执行上下文** (2)：teammateContext、toolUseContext
- **生命周期** (1)：abortController
- **模型与 prompt** (3)：model / systemPrompt / systemPromptMode
- **权限** (2)：allowedTools / allowPermissionPrompts
- **追溯** (1)：invokingRequestId

这种"参数对象越来越胖"是工业代码常态——每加一个 feature 就加一个 optional 字段。Config 类型用 `?:` 大量 optional 让旧代码不需要修改也能调用。

## 7.3 InProcessTeammateTask 状态字段（在 AppState.tasks 里）

`InProcessTeammateTask/types.ts` 定义的 task 状态（推断字段）：

```ts
type InProcessTeammateTaskState = {
  type: 'in_process_teammate'
  id: string                                 // 同 taskId
  status: 'running' | 'idle' | 'killed' | 'completed'
  identity: TeammateIdentity
  isIdle: boolean                            // 当前 turn 结束 vs 处理中
  abortController: AbortController           // 整个 teammate 的 lifecycle abort
  workAbortController?: AbortController      // 当前 turn 的 work abort (per-turn)
  shutdownRequested?: boolean                // leader 发过 shutdown_request
  messages: Message[]                        // 消息历史 (UI 显示)
  onIdleCallbacks: Array<() => void>         // 等 teammate idle 的回调列表
  // ... 等
}
```

3 个**独立 abortController** 的设计：

| Controller | 作用 |
|------------|------|
| `task.abortController` | 整个 teammate lifecycle——kill 时调用 |
| `task.workAbortController` | 当前 turn 的工作——ESC 时调用 (per-turn) |
| `runInProcessTeammate(config).abortController` | 同 `task.abortController`（同一对象） |

**lifecycle vs work 分离**：用户按 ESC 应该只停当前 turn，不应该终结整个 teammate。如果两者用同一个 controller，ESC 会让 teammate 进程结束——bad UX。

注释（`inProcessRunner.ts:1053-1056`）：

> Create a per-turn abort controller for this iteration. This allows Escape to stop current work without killing the whole teammate. The lifecycle abortController still kills the whole teammate if needed.

## 7.4 startInProcessTeammate —— fire-and-forget 入口

`inProcessRunner.ts:1544`：

```ts
export function startInProcessTeammate(config: InProcessRunnerConfig): void {
  // 实际实现 (推断): 用 runWithTeammateContext 包装 runInProcessTeammate
  runWithTeammateContext(config.teammateContext, async () => {
    try {
      await runInProcessTeammate(config)
    } catch (err) {
      logError(err)
    }
  })
}
```

返回 `void`——**没有 await 的入口**。注释说明这是 fire-and-forget——caller 立即返回，runner 在后台跑。

[02 Teammate 身份](./02-Teammate身份与上下文.md) 讲过 `runWithTeammateContext` 是 AsyncLocalStorage 入口——把 teammate 的 context 包装到所有异步派生里。

## 7.5 runInProcessTeammate —— 主循环全貌

`inProcessRunner.ts:883-1543` 的 660+ 行主函数。简化的结构：

```ts
export async function runInProcessTeammate(
  config: InProcessRunnerConfig,
): Promise<InProcessRunnerResult> {
  const { identity, taskId, prompt, ..., abortController } = config

  // 0. 构造 AgentContext (analytics)
  const agentContext: AgentContext = { ... }

  // 1. 构造 system prompt (default / replace / append 三种模式)
  let teammateSystemPrompt: string
  if (systemPromptMode === 'replace' && systemPrompt) {
    teammateSystemPrompt = systemPrompt
  } else {
    const fullSystemPromptParts = await getSystemPrompt(tools, model, undefined, mcpClients)
    const systemPromptParts = [
      ...fullSystemPromptParts,
      TEAMMATE_SYSTEM_PROMPT_ADDENDUM,  // teammate 专用补充
    ]
    if (agentDefinition) {
      systemPromptParts.push(`\n# Custom Agent Instructions\n${agentDefinition.getSystemPrompt()}`)
    }
    if (systemPromptMode === 'append' && systemPrompt) {
      systemPromptParts.push(systemPrompt)
    }
    teammateSystemPrompt = systemPromptParts.join('\n')
  }

  // 2. 构造 resolvedAgentDefinition (注入必要工具)
  const resolvedAgentDefinition: CustomAgentDefinition = {
    agentType: identity.agentName,
    getSystemPrompt: () => teammateSystemPrompt,
    tools: agentDefinition?.tools
      ? [...new Set([...agentDefinition.tools, SEND_MESSAGE, TEAM_CREATE, TASK_CREATE, ...])]
      : ['*'],
    permissionMode: 'default',  // teammate 总是 default mode
  }

  // 3. 初始化状态
  const allMessages: Message[] = []
  const wrappedInitialPrompt = formatAsTeammateMessage('team-lead', prompt, undefined, description)
  let currentPrompt = wrappedInitialPrompt
  let shouldExit = false

  // 4. 试着 claim 第一个任务
  await tryClaimNextTask(identity.parentSessionId, identity.agentName)

  try {
    // 把初始 prompt 加进 task.messages 让 UI 显示
    updateTaskState(taskId, task => ({
      ...task,
      messages: appendCappedMessage(task.messages, createUserMessage({ content: wrappedInitialPrompt })),
    }), setAppState)

    // per-teammate content replacement state (跨 turn 持久)
    let teammateReplacementState = toolUseContext.contentReplacementState
      ? createContentReplacementState()
      : undefined

    // 5. 主循环
    while (!abortController.signal.aborted && !shouldExit) {
      const currentWorkAbortController = createAbortController()
      updateTaskState(taskId, task => ({ ..., workAbortController: currentWorkAbortController }), setAppState)

      // 5a. 跑一轮 runAgent
      const turnResult = await runAgent({
        identity,
        prompt: currentPrompt,
        messages: allMessages,
        agentDefinition: resolvedAgentDefinition,
        abortController: currentWorkAbortController,
        canUseTool: createInProcessCanUseTool(...),
        // ... 等
      })

      // 5b. 累积消息
      allMessages.push(...turnResult.messages)

      // 5c. 更新 task.isIdle = true
      updateTaskState(taskId, task => ({ ..., isIdle: true, workAbortController: undefined }), setAppState)

      // 5d. 触发 onIdleCallbacks
      task.onIdleCallbacks?.forEach(cb => cb())

      // 5e. 发 idle_notification 给 lead
      await sendIdleNotification(identity.agentName, identity.color, identity.teamName, {
        idleReason: turnResult.idleReason,
        completedTaskId: turnResult.completedTaskId,
        completedStatus: turnResult.completedStatus,
        summary: turnResult.summary,
      })

      // 5f. 等下一个 prompt 或 shutdown
      const waitResult = await waitForNextPromptOrShutdown(taskId, identity, abortController, setAppState)

      switch (waitResult.kind) {
        case 'next_prompt':
          currentPrompt = waitResult.prompt
          break
        case 'shutdown':
          shouldExit = true
          break
        case 'abort':
          shouldExit = true
          break
      }

      // 5g. 更新 task.isIdle = false (开始下一 turn)
      if (!shouldExit) {
        updateTaskState(taskId, task => ({ ..., isIdle: false }), setAppState)
      }
    }

    return { success: true, messages: allMessages }
  } catch (err) {
    // 错误处理 ...
    return { success: false, error, messages: allMessages }
  }
}
```

7 步主循环：

| 步 | 动作 |
|---|------|
| 5a | 跑 runAgent 一个 turn |
| 5b | 累积消息到 allMessages |
| 5c | 标记 task.isIdle = true |
| 5d | 触发 onIdleCallbacks |
| 5e | 发 idle_notification 给 lead |
| 5f | 等下一个 prompt 或 shutdown |
| 5g | 下个 turn 前重置 isIdle = false |

整个 teammate 的生命周期 = 这个 while loop 跑直到 `shouldExit` 或 `abortController.signal.aborted`。

## 7.6 system prompt 三种模式

`runInProcessTeammate:923-970` 构造 system prompt 时支持 3 种模式：

| `systemPromptMode` | 行为 |
|--------------------|------|
| `'replace'` + `systemPrompt` | **完全替换**默认 system prompt |
| `'append'` + `systemPrompt` | 默认 + teammate addendum + custom + 用户附加 |
| `'default'` 或缺省 | 默认 + teammate addendum + custom |

`TEAMMATE_SYSTEM_PROMPT_ADDENDUM` 是 teammate **额外**的 system prompt 内容（在 `utils/swarm/teammatePromptAddendum.ts`，18 行）——告诉模型"你是 teammate、你属于团队 X、你能用 SendMessage 等工具"。

## 7.7 注入必要工具

`runInProcessTeammate:982-995`：

```ts
tools: agentDefinition?.tools
  ? [
      ...new Set([
        ...agentDefinition.tools,
        SEND_MESSAGE_TOOL_NAME,
        TEAM_CREATE_TOOL_NAME,
        TEAM_DELETE_TOOL_NAME,
        TASK_CREATE_TOOL_NAME,
        TASK_GET_TOOL_NAME,
        TASK_LIST_TOOL_NAME,
        TASK_UPDATE_TOOL_NAME,
      ]),
    ]
  : ['*'],
```

**即使 agent 定义里限制了工具列表**，仍**强制注入** 7 个 team-essential 工具：

- SendMessage：回应 shutdown / 发消息给其它 teammate；
- TeamCreate / TeamDelete：管理团队（理论上 teammate 也可以）；
- TaskCreate / TaskGet / TaskList / TaskUpdate：协调任务列表。

注释：

> Inject team-essential tools so teammates can always respond to shutdown requests, send messages, and coordinate via the task list, even with explicit tool lists

**协议必需 > 用户限制**——即使用户设了"researcher 只能用 Read 和 Grep"，也得让它能 SendMessage（否则没法响应 shutdown）。

这是 **协议级 hard 注入** 的设计——某些能力是 swarm 协议的前提条件，不能被绕过。

## 7.8 permissionMode 强制 default

```ts
const resolvedAgentDefinition: CustomAgentDefinition = {
  // ...
  permissionMode: 'default',
}
```

注释（`inProcessRunner.ts:973-974`）：

> IMPORTANT: Set permissionMode to 'default' so teammates always get full tool access regardless of the leader's permission mode.

**lead 的 permission mode 不传递给 teammate**——teammate 总是用 `default` mode。

为什么？因为 lead 可能在 `plan` mode 或 `auto` mode 之类——这些 mode 限制工具行为。但 teammate 的工具调用通过 [08 Permission Sync](./08-Permission-Sync.md) 全部 forward 给 leader 决定——teammate 自己的 mode 没用，反而可能导致工具被本地拒绝（无法 forward）。

`default` mode 是个 **passthrough mode**——所有调用都问权限（实际通过 mailbox 问 leader）。

## 7.9 tryClaimNextTask —— 任务自动认领

`inProcessRunner.ts:624-657`：

```ts
async function tryClaimNextTask(
  taskListId: string,
  agentName: string,
): Promise<string | undefined> {
  try {
    const tasks = await listTasks(taskListId)
    const availableTask = findAvailableTask(tasks)
    if (!availableTask) return undefined

    const result = await claimTask(taskListId, availableTask.id, agentName)
    if (!result.success) {
      logForDebugging(`[inProcessRunner] Failed to claim task #${availableTask.id}: ${result.reason}`)
      return undefined
    }

    await updateTask(taskListId, availableTask.id, { status: 'in_progress' })
    return formatTaskAsPrompt(availableTask)
  } catch (err) {
    return undefined
  }
}
```

`findAvailableTask` 找 **没人 owns、pending、依赖已完成** 的 task。teammate 启动时自动 claim 一个——不需要 lead 显式分配。

注释（`inProcessRunner.ts:1015-1019`）：

> Try to claim an available task immediately so the UI can show activity from the very start. The idle loop handles claiming for subsequent tasks.

启动时立刻 claim 让 UI **立即有活动显示**——避免 teammate 启动后"看起来停着"。后续任务在 idle loop 里 claim。

`formatTaskAsPrompt` 把 task 转成 prompt：

```ts
function formatTaskAsPrompt(task: Task): string {
  let prompt = `Complete all open tasks. Start with task #${task.id}: \n\n ${task.subject}`
  if (task.description) prompt += `\n\n${task.description}`
  return prompt
}
```

简单字符串拼接——把任务变成"接下来该做这个"的 prompt。

## 7.10 waitForNextPromptOrShutdown —— 3 种状态等待

`inProcessRunner.ts:662-687` 的 `WaitResult` 类型：

```ts
type WaitResult =
  | { kind: 'next_prompt'; prompt: string }
  | { kind: 'shutdown'; approved: boolean }
  | { kind: 'abort' }
```

每个 turn 结束后 teammate 进入"等待"状态。可能的转换：

1. **`next_prompt`**：mailbox 收到新消息（普通 message 或 plan_approval_response）→ 处理这个；
2. **`shutdown`**：收到 shutdown_request 且 teammate 同意 → 退出 loop；
3. **`abort`**：lifecycle abortController 被触发（kill）→ 退出 loop。

`waitForNextPromptOrShutdown` 在 `689-880` 行，约 200 行。实现轮询：

```ts
async function waitForNextPromptOrShutdown(...): Promise<WaitResult> {
  while (!abortController.signal.aborted) {
    // 1. 检查 mailbox
    const messages = await readUnreadMessages(agentName, teamName)

    for (const msg of messages) {
      // 检查协议消息
      if (isShutdownRequest(msg.text)) {
        const shouldApprove = await decideShutdown(msg.text, ...)
        if (shouldApprove) {
          await sendShutdownApproved(...)
          return { kind: 'shutdown', approved: true }
        } else {
          await sendShutdownRejected(...)
          continue  // 继续等下一个消息
        }
      }

      if (isPlanApprovalResponse(msg.text)) {
        // 处理 plan 批准
        const planResp = isPlanApprovalResponse(msg.text)
        if (planResp.approved) {
          // 退出 plan mode, 继续 implementation
        }
      }

      // 普通消息 → 当下一个 prompt
      await markMessageAsRead(...)
      return {
        kind: 'next_prompt',
        prompt: formatAsTeammateMessage(msg.from, msg.text, msg.color, msg.summary),
      }
    }

    // 2. 试着 claim 任务
    const taskPrompt = await tryClaimNextTask(taskListId, agentName)
    if (taskPrompt) {
      return { kind: 'next_prompt', prompt: taskPrompt }
    }

    // 3. 都没有 → 等一会儿再轮询
    await sleep(PERMISSION_POLL_INTERVAL_MS)  // 500ms
  }

  return { kind: 'abort' }
}
```

**3 个等待来源**：mailbox 新消息、可认领的 task、abort 信号。轮询间隔 500ms。

这是 **典型的 actor 主循环**——一直等事件、处理事件、循环。

## 7.11 PERMISSION_POLL_INTERVAL_MS = 500

```ts
const PERMISSION_POLL_INTERVAL_MS = 500
```

500ms 是个**人因学合理值**——

- 用户感知不到明显延迟（<1s 不会觉得卡）；
- 文件 IO 频率不会过高（500ms 间隔每分钟 120 次 fstat 而已）；
- 给 worker 留出处理时间（不会一直 busy poll）。

但仍是**轮询**——不是事件驱动。如果有更好的 IPC 机制（如 SIGUSR1 通知 mailbox 更新）能避免轮询。Claude Code 选择简洁的轮询设计。

## 7.12 createInProcessCanUseTool —— 权限决策注入

`inProcessRunner.ts:128+`：

```ts
function createInProcessCanUseTool(
  identity: TeammateIdentity,
  toolUseContext: ToolUseContext,
  abortController: AbortController,
  allowedTools?: string[],
  allowPermissionPrompts?: boolean,
): CanUseToolFn {
  return async (toolName, input, suggestions) => {
    // 1. 检查 allowedTools 白名单
    if (allowedTools && !allowedTools.includes(toolName) && !allowedTools.includes('*')) {
      // 工具不在白名单 → deny
      return { behavior: 'deny', message: ... }
    }

    // 2. 不允许 prompt 的工具 + 未白名单 → auto deny
    if (allowPermissionPrompts === false && !isAutoAllowed(toolName)) {
      return { behavior: 'deny', ... }
    }

    // 3. forward 到 leader (通过 mailbox)
    const requestId = generateRequestId('permission', identity.agentId)
    await writeToMailbox(TEAM_LEAD_NAME, {
      text: JSON.stringify({
        type: 'permission_request',
        request_id: requestId,
        agent_id: identity.agentId,
        tool_name: toolName,
        // ...
      }),
    }, identity.teamName)

    // 4. 轮询 mailbox 等 leader response
    const response = await waitForPermissionResponse(requestId, identity, abortController)
    return response  // { behavior: 'allow', updated_input }
  }
}
```

teammate 跑工具时不本地决策——通过 mailbox 把请求**转给 leader**，leader 弹窗给用户，response 通过 mailbox 回来。

**完整实现在 [08 Permission Sync](./08-Permission-Sync.md)** 展开。本篇只需要知道 runner 通过 `createInProcessCanUseTool` 注入这套机制。

## 7.13 cross-turn content replacement state

`inProcessRunner.ts:1035-1045`：

```ts
// Per-teammate content replacement state. The while-loop below calls
// runAgent repeatedly over an accumulating `allMessages` buffer (which
// carries FULL original tool result content, not previews — query() yields
// originals, enforcement is non-mutating). Without persisting state across
// iterations, each call gets a fresh empty state from createSubagentContext
// and makes holistic replace-globally-largest decisions, diverging from
// earlier iterations' incremental frozen-first decisions → wire prefix
// differs → cache miss. Gated on parent to inherit feature-flag-off.
let teammateReplacementState = toolUseContext.contentReplacementState
  ? createContentReplacementState()
  : undefined
```

这段注释揭示一个**深 cache 问题**：

- runAgent 在 while loop 里被多次调用、共享 `allMessages` buffer；
- 每次调用都做 content replacement（把过大的 tool result 替换成 preview）；
- **如果每次都从空状态开始**——同样的 tool result 在第 1/2/3 轮可能被替换成不同长度的 preview（因为 "全局最大" 是相对 buffer 当前状态）；
- 不同长度 = wire 不同 = prompt cache miss；
- **必须跨 turn 持久化 replacement state**——让"已 frozen 的替换"在后续保持不变。

这是 **prompt caching 与多轮执行的细微交互**——同样的设计陷阱 [messages-pipeline 03](../messages-pipeline/03-normalize主循环与三路分发.md) 提过 normalize 的 cache stability。

## 7.14 updateTaskState —— AppState 更新 helper

`inProcessRunner.ts:519-541`：

```ts
function updateTaskState(
  taskId: string,
  updater: (task: InProcessTeammateTaskState) => InProcessTeammateTaskState,
  setAppState: SetAppStateFn,
): void {
  setAppState(prev => {
    const task = prev.tasks[taskId]
    if (!task || task.type !== 'in_process_teammate') {
      return prev
    }
    const updated = updater(task)
    if (updated === task) return prev  // 优化: 无变化不创建新 state

    return { ...prev, tasks: { ...prev.tasks, [taskId]: updated } }
  })
}
```

3 个细节：

### 1. type guard

```ts
if (!task || task.type !== 'in_process_teammate') return prev
```

如果 task 已经被删除或类型变了 → 不动 state。这种**防御性 guard** 让并发 setAppState 不会破坏数据。

### 2. 引用相等优化

```ts
const updated = updater(task)
if (updated === task) return prev
```

如果 updater 返回同一对象 → 不创建新 state。这避免 React 因为 reference change 重渲染。

### 3. immutable update

```ts
return { ...prev, tasks: { ...prev.tasks, [taskId]: updated } }
```

shallow copy + override——immutable 风格。深层（task 内的 messages 等）由 updater 自己负责。

## 7.15 几个隐性设计判断

### 1. 3 个独立 abortController

lifecycle / work / per-turn 分离。让 ESC 只停当前 turn、kill 终结整个 teammate、abort 全局停止——3 个独立信号。这是个**精细的生命周期控制**。

### 2. 协议必需工具强制注入

即使 agent 限制了工具，team-essential 工具（SendMessage / Task* 等）也必须可用——否则破坏协议。**协议约束 > 用户配置**。

### 3. permissionMode 强制 default

teammate 不继承 lead 的 mode——所有权限通过 mailbox forward 到 leader 决定。**本地 mode 没意义**——直接 default。

### 4. 启动时立刻 claim 任务

避免 teammate 启动后看起来停着——UI 立即有活动。这种**zero-delay activity feedback** 是 UX 关键。

### 5. 500ms 轮询 mailbox

不用事件驱动 IPC——简单的轮询。500ms 兼顾感知和资源消耗。**简洁性 > 性能**。

### 6. cross-turn replacement state 持久

避免每轮新建空 state 导致 prompt cache miss。这种**对 cache 友好的状态管理**是工业代码细节。

### 7. setAppState 引用相等优化

避免无变化触发 React 重渲染。**精细的 React 性能优化**——React 用引用比较判定是否需要重新渲染。

### 8. fire-and-forget 入口

`startInProcessTeammate` 不 await——teammate 是 actor 不是 RPC。

### 9. 注释里写 cache miss 原因

第 7.13 节那段长注释直接解释 "wire prefix differs → cache miss"——这种**把性能优化背后的原因写进注释**是 Claude Code 工程文化。

## 7.16 1552 行复杂度的来源

回顾本篇覆盖的内容：

- AsyncLocalStorage 隔离入口（runWithTeammateContext）
- system prompt 三种模式构造
- 协议必需工具强制注入
- 任务自动认领（tryClaimNextTask）
- 主 while loop（runInProcessTeammate）
- 3 个独立 abortController 管理
- mailbox 轮询 + 协议消息分发（waitForNextPromptOrShutdown）
- permission forward 到 leader（createInProcessCanUseTool）
- cross-turn content replacement state
- idle_notification 自动发送
- task state immutable update（updateTaskState）
- shutdown_request 决策与响应
- plan_approval_response 处理
- runAgent 调用与消息累积
- onIdleCallbacks 回调列表
- 错误处理与日志

每一项都不算特别复杂，但加起来 1552 行——**协调多个独立子系统**就是这个数量级。

## 7.17 与其它专题串联

- **[02 Teammate 身份](./02-Teammate身份与上下文.md)**：本篇用 `runWithTeammateContext` 进入 AsyncLocalStorage 作用域；
- **[04 In-process backend](./04-In-process-backend.md)**：InProcessBackend.spawn() 调本篇的 startInProcessTeammate；
- **[05 消息系统](./05-消息系统与SendMessage.md)**：本篇通过 writeToMailbox / readMailbox 跟其它 teammate 通信；
- **[06 协议消息](./06-协议消息状态机.md)**：本篇的 waitForNextPromptOrShutdown 处理 shutdown_request / plan_approval；
- **[08 Permission Sync](./08-Permission-Sync.md)**：本篇的 createInProcessCanUseTool 是 worker 端的 permission forward；
- **[agent 06 runAgent](../agent/06-runAgent核心机制.md)**：本篇调 runAgent 跑每个 turn，相当于 sub-agent 模式的复用。

## 7.18 小结

- inProcessRunner 1552 行实现 in-process teammate 的完整生命周期；
- 主入口 `startInProcessTeammate` fire-and-forget，内部包 runWithTeammateContext；
- 16 字段的 InProcessRunnerConfig 反映多 feature 配置点；
- 3 个独立 abortController：lifecycle / work / per-turn；
- system prompt 三种模式：replace / append / default + teammate addendum；
- 7 个 team-essential 工具强制注入即使被 agent 限制；
- permissionMode 强制 default——所有权限走 leader forward；
- 启动时立刻 claim 任务避免 UI 看起来停滞；
- 主 while loop 7 步：runAgent → 累积消息 → mark idle → 触发回调 → 发 idle_notification → 等下个 prompt/shutdown → 重置 idle；
- waitForNextPromptOrShutdown 500ms 轮询 mailbox + task list；
- cross-turn content replacement state 持久化避免 prompt cache miss；
- updateTaskState 的 immutable 风格 + 引用相等优化避免 React 无效重渲染；
- 注释里写下"wire prefix differs → cache miss"这种性能原因。

下一篇 → [08 Permission Sync](./08-Permission-Sync.md)
