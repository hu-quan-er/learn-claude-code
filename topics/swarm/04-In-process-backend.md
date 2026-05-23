# 04 In-process backend

> 没有 tmux 也没有 iTerm2 时怎么办——InProcessBackend 339 行实现 TeammateExecutor 接口，但没有真 pane。本篇拆它和 pane backend 的本质差异、它如何当 fallback、它如何利用 AsyncLocalStorage 在同进程跑多个 Claude 实例。

## 4.1 接口差异：实现 TeammateExecutor 而不是 PaneBackend

回顾 [03 Pane backend](./03-Pane-backend-tmux-iterm2.md) 的 `PaneBackend` 接口 —— 11 个方法都和"真 pane"相关。但 InProcessBackend 没有 pane！

`backends/InProcessBackend.ts:38`：

```ts
export class InProcessBackend implements TeammateExecutor {
  readonly type = 'in-process' as const
  // ...
}
```

它实现的是 `TeammateExecutor` 接口而不是 `PaneBackend`：

```ts
// backends/types.ts:279-300
export type TeammateExecutor = {
  readonly type: BackendType
  isAvailable(): Promise<boolean>
  spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult>
  sendMessage(agentId: string, message: TeammateMessage): Promise<void>
  terminate(agentId: string, reason?: string): Promise<boolean>
  kill(agentId: string): Promise<boolean>
  isActive(agentId: string): Promise<boolean>
}
```

6 个方法——**关心 teammate 生命周期，不关心 pane 操作**。

为什么有两个接口？

- **`PaneBackend`**：低层 pane 操作（split / send-keys / kill-pane / hide-pane）；
- **`TeammateExecutor`**：高层 teammate 生命周期（spawn / sendMessage / terminate / kill / isActive）。

pane backend 包装一层适配器（`PaneBackendExecutor.ts`）变成 `TeammateExecutor`。in-process backend 直接实现 `TeammateExecutor`——因为它没有 pane 概念，不需要适配。

```
┌──────────────────────────────────────┐
│ 上层代码 (spawnMultiAgent, runner)    │
│ 通过 TeammateExecutor 接口调用        │
└────────────────┬─────────────────────┘
                 │
        ┌────────┴────────┐
        ▼                 ▼
┌───────────────┐  ┌─────────────────┐
│ InProcess     │  │ PaneBackend     │
│ Backend       │  │ Executor (适配) │
│ (直接实现)    │  │                  │
└───────────────┘  └────────┬────────┘
                            │
                            ▼
                    ┌──────────────────┐
                    │ PaneBackend       │
                    │ (tmux/iTerm2)    │
                    └──────────────────┘
```

`PaneBackendExecutor` 把 PaneBackend 的低层操作适配成 TeammateExecutor 的高层语义。**上层永远调 `TeammateExecutor`**——不区分底下是 in-process 还是 pane。

## 4.2 三大本质差异

InProcessBackend 和 PaneBackend 的不同在 3 个维度：

| 维度 | PaneBackend (tmux/iTerm2) | InProcessBackend |
|------|--------------------------|-------------------|
| **进程模型** | 每个 teammate 是独立进程 | 所有 teammate 在 lead 进程内 |
| **隔离机制** | OS 进程隔离 | AsyncLocalStorage |
| **生命周期** | spawn 子进程 / kill-pane | start agent loop / AbortController |
| **资源** | 各自独立 API client、MCP 连接 | 与 lead **共享** API client、MCP 连接 |
| **可视性** | 用户在 pane 里看到 teammate 输出 | 用户**看不到独立 pane**——teammate 输出走专门 UI |
| **崩溃影响** | teammate 崩溃不影响 lead | teammate 崩溃可能拖垮 lead 进程 |

文件头注释（`InProcessBackend.ts:25-32`）：

> Unlike pane-based backends (tmux/iTerm2), in-process teammates run in the same Node.js process with isolated context via AsyncLocalStorage. They:
> - Share resources (API client, MCP connections) with the leader
> - Communicate via file-based mailbox (same as pane-based teammates)
> - Are terminated via AbortController (not kill-pane)

3 个特点：**共享资源、用同样的 mailbox（即使在同进程也走文件）、用 AbortController 而不是 kill-pane**。

## 4.3 为什么文件 mailbox 即使在同进程也走文件

最有意思的设计判断：**in-process teammate 和 lead 在同进程，但通信仍然走 file-based mailbox**（`teammateMailbox.ts`）。

为什么不用内存队列直接通信？

```ts
// InProcessBackend.ts:147-180
async sendMessage(agentId: string, message: TeammateMessage): Promise<void> {
  const parsed = parseAgentId(agentId)
  if (!parsed) throw new Error(`Invalid agentId format: ${agentId}.`)
  const { agentName, teamName } = parsed

  // Write to file-based mailbox
  await writeToMailbox(
    agentName,
    {
      text: message.text,
      from: message.from,
      color: message.color,
      timestamp: message.timestamp ?? new Date().toISOString(),
    },
    teamName,
  )
}
```

注释（`InProcessBackend.ts:148-149`）：

> All teammates use file-based mailboxes for simplicity.

**为了一致性**——in-process 走文件 mailbox 让代码路径和 pane backend 完全一样。优势：

1. **同一套消息持久化**——in-process teammate 崩溃后 lead 重启能 resume 未读消息；
2. **同一套 attachment surfacer**——messages-pipeline 不需要为 in-process 写特殊路径；
3. **同一套 lockfile 并发控制**——不需要单独的内存并发原语；
4. **同一套 audit trail**——所有消息都写到磁盘可以事后审查。

代价：

- in-process 时多一次文件 IO（毫秒级）；
- 文件 mailbox 的并发性能上限低于内存队列。

**简洁性 > 性能优化**——大部分场景下 teammate 间消息频率不高（人类级别），文件 IO 完全够。

## 4.4 isAvailable 永远 true

```ts
async isAvailable(): Promise<boolean> {
  return true
}
```

注释明确："In-process backend is always available (no external dependencies)."

这让 InProcessBackend 成为**永远可用的 fallback**——detection 失败、tmux / iTerm2 都不可用时，至少 in-process 能工作。

[02 Teammate 身份与上下文](./02-Teammate身份与上下文.md) 提过 AsyncLocalStorage 让 in-process 模型成为可能——这里我们看到它**默认 always-on**。

## 4.5 ToolUseContext 注入

```ts
private context: ToolUseContext | null = null

setContext(context: ToolUseContext): void {
  this.context = context
}
```

注释（`InProcessBackend.ts:34-37`）：

> IMPORTANT: Before spawning, call setContext() to provide the ToolUseContext needed for AppState access. This is intended for use via the TeammateExecutor abstraction (getTeammateExecutor() in registry.ts).

**InProcessBackend 不是无状态的**——它需要 ToolUseContext 来访问 AppState（任务列表、teamContext 等）。在 `registry.ts:getTeammateExecutor()` 里通过 `setContext()` 注入。

为什么 pane backend 不需要 ToolUseContext？因为 pane backend 通过 spawn 子进程，子进程有自己的 AppState。in-process backend **在 lead 的 AppState 里管理 teammate 任务**（`appState.tasks[taskId]`），所以需要 setAppState 能力。

`spawn()` 的第一行 guard：

```ts
async spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult> {
  if (!this.context) {
    return {
      success: false,
      agentId: `${config.name}@${config.teamName}`,
      error: 'InProcessBackend not initialized. Call setContext() before spawn().',
    }
  }
  // ...
}
```

没 context → 立刻返回 spawn 失败。**显式 fail-loud** 而不是隐式 undefined-crash。

## 4.6 spawn 流程

```ts
async spawn(config: TeammateSpawnConfig): Promise<TeammateSpawnResult> {
  if (!this.context) return { success: false, ... }

  // 1. 调 spawnInProcessTeammate (utils/swarm/spawnInProcess.ts)
  const result = await spawnInProcessTeammate(
    {
      name: config.name,
      teamName: config.teamName,
      prompt: config.prompt,
      color: config.color,
      planModeRequired: config.planModeRequired ?? false,
    },
    this.context,
  )

  // 2. spawn 成功 → 启动 agent loop (fire-and-forget)
  if (
    result.success &&
    result.taskId &&
    result.teammateContext &&
    result.abortController
  ) {
    startInProcessTeammate({
      identity: { ... },
      taskId: result.taskId,
      prompt: config.prompt,
      teammateContext: result.teammateContext,
      // Strip messages: ...
      toolUseContext: { ...this.context, messages: [] },
      abortController: result.abortController,
      model: config.model,
      systemPrompt: config.systemPrompt,
      // ...
    })
  }

  return { success, agentId, taskId, abortController, error }
}
```

两步：

### 步骤 1：spawnInProcessTeammate —— 创建 context + 注册任务

`utils/swarm/spawnInProcess.ts`（328 行）做的事：

1. 调 `createTeammateContext(...)` 创建 AsyncLocalStorage 用的 context；
2. 创建**独立**的 AbortController（不绑 parent）；
3. 在 `AppState.tasks` 里注册 `in_process_teammate` 类型的任务；
4. 写 teammate 到团队 config（`teamFile.members.push(...)`）；
5. 返回 `{ agentId, taskId, teammateContext, abortController }`。

### 步骤 2：startInProcessTeammate —— fire-and-forget 启动 agent loop

`startInProcessTeammate(...)` 来自 `inProcessRunner.ts` 1552 行——它是 in-process teammate 的"runtime"。

注意调用方式：**没 await**！

```ts
startInProcessTeammate({...})

logForDebugging(`[InProcessBackend] Started agent execution for ${result.agentId}`)
```

**fire-and-forget**——启动 teammate 后立刻返回 spawn 结果。teammate 在后台继续跑。

为什么？因为 teammate 是长期运行的 actor——`spawn()` 不该等 teammate 跑完。lead 调 spawn 拿到 agentId 后就能开始往那发消息了。

### 关键细节：剥离 messages

```ts
// Strip messages: the teammate never reads toolUseContext.messages
// (runAgent overrides it via createSubagentContext). Passing the
// parent's conversation would pin it for the teammate's lifetime.
toolUseContext: { ...this.context, messages: [] },
```

注释明确：teammate **不会**用 `toolUseContext.messages`——它通过 `createSubagentContext` 自己生成新的 messages 上下文（[agent 06](../agent/06-runAgent核心机制.md) 讲过）。

如果原样传 parent 的 messages，会**钉住** parent 当前的对话历史不被 GC——teammate 长期运行的话内存占用爆掉。

**显式置空 + 注释解释**是个清晰的设计——不是"忘记传"，而是"故意不传，理由是 X"。

## 4.7 sendMessage —— 与 pane backend 完全相同

```ts
async sendMessage(agentId: string, message: TeammateMessage): Promise<void> {
  const parsed = parseAgentId(agentId)
  if (!parsed) throw new Error(...)

  const { agentName, teamName } = parsed
  await writeToMailbox(agentName, { ... }, teamName)
}
```

调用 `writeToMailbox` 写到文件——和 pane backend 走的是**同一个函数**。

这就是 4.3 节讲的"统一通信路径"。两种 backend 唯一不同是 spawn / terminate / kill，sendMessage 完全一样。

## 4.8 terminate —— 优雅关闭流程

```ts
async terminate(agentId: string, reason?: string): Promise<boolean> {
  if (!this.context) return false

  const state = this.context.getAppState()
  const task = findTeammateTaskByAgentId(agentId, state.tasks)
  if (!task) return false

  // 已经发过 shutdown request 就不再发
  if (task.shutdownRequested) return true

  // 生成确定性 request ID
  const requestId = `shutdown-${agentId}-${Date.now()}`

  // 构造 shutdown 协议消息
  const shutdownRequest = createShutdownRequestMessage({
    requestId,
    from: 'team-lead',
    reason,
  })

  // 通过 mailbox 发送
  const teammateAgentName = task.identity.agentName
  await writeToMailbox(
    teammateAgentName,
    {
      from: 'team-lead',
      text: jsonStringify(shutdownRequest),
      timestamp: new Date().toISOString(),
    },
    task.identity.teamName,
  )

  // 标记任务 shutdown requested
  requestTeammateShutdown(task.id, this.context.setAppState)

  return true
}
```

`terminate` 是**优雅关闭**——发 shutdown_request 协议消息给 teammate。teammate 处理这个消息后自己决定是否同意（[06 协议消息状态机](./06-协议消息状态机.md) 详述）。

3 个细节：

### 1. 去重——已发过就不再发

```ts
if (task.shutdownRequested) {
  return true
}
```

**幂等性**——重复 terminate 同一个 agent 不发多份 request。

### 2. requestId 用 `shutdown-${agentId}-${timestamp}`

确定性 ID 让 teammate 在 response 里 echo 时 lead 能匹配请求。详见 [06](./06-协议消息状态机.md)。

### 3. 通过 mailbox 而不是直接调函数

虽然 teammate 在同进程，但 shutdown_request 仍然**走 mailbox 文件**——和 sendMessage 一样的路径。这又是 4.3 节"统一通信"的体现。

`requestTeammateShutdown(task.id, setAppState)` 只是**额外**在 AppState 标记一下"已发过 request"——用于 UI 显示和去重。真正的传递走 mailbox。

## 4.9 kill —— 强制终止

```ts
async kill(agentId: string): Promise<boolean> {
  if (!this.context) return false

  const state = this.context.getAppState()
  const task = findTeammateTaskByAgentId(agentId, state.tasks)
  if (!task) return false

  const killed = killInProcessTeammate(task.id, this.context.setAppState)
  return killed
}
```

调 `killInProcessTeammate(taskId, setAppState)`——它在 `spawnInProcess.ts` 里：

1. 找到 task 的 `abortController`；
2. 调 `abortController.abort()`——取消所有正在跑的异步操作；
3. 更新 AppState 把 task status 设为 `'killed'`；
4. UI 显示 teammate 已终止。

**通过 AbortController 关 teammate**——不是 SIGKILL 一个进程，而是让 teammate 内部的所有 await 都收到 abort 信号。

这是 [02 Teammate 身份](./02-Teammate身份与上下文.md) 提过的"teammate 的 AbortController 独立于 parent"——独立意味着 lead 可以单独 abort 某个 teammate，不影响别的 teammate / 不影响 lead 自己。

## 4.10 isActive —— 双重判定

```ts
async isActive(agentId: string): Promise<boolean> {
  if (!this.context) return false

  const state = this.context.getAppState()
  const task = findTeammateTaskByAgentId(agentId, state.tasks)
  if (!task) return false

  const isRunning = task.status === 'running'
  const isAborted = task.abortController?.signal.aborted ?? true

  return isRunning && !isAborted
}
```

**两个独立信号都得满足**：

- `task.status === 'running'`：AppState 里标记的状态；
- `!task.abortController.signal.aborted`：abort signal 没被触发。

为什么要双重？因为这两个状态可能**短暂不一致**：
- `abortController.abort()` 触发瞬间，task.status 还是 'running'（还没更新）；
- 反过来，task.status 已经 'completed' 但 abortController 还未 cleanup。

**两个都要 OK 才算 active**——保守。

注意默认值 `?? true` 让 missing abortController 也算 aborted——更保守。

## 4.11 与 InProcess Runner 的关系

InProcessBackend 是个**外壳**——真正干活的是 `inProcessRunner.ts` (1552 行) 里的 `startInProcessTeammate`。

InProcessBackend 的责任：
- 实现 TeammateExecutor 接口；
- 调用 spawnInProcessTeammate 创建 context；
- 调用 startInProcessTeammate 启动 runner；
- 通过 setAppState 更新 task 状态。

InProcessRunner 的责任（[07 In-process Runner](./07-In-process-Runner.md) 展开）：
- 在 AsyncLocalStorage context 里跑完整的 Claude agent loop；
- 监听 mailbox 接收消息；
- 处理 shutdown / plan_approval 协议；
- 与 leader 同步权限请求。

**接口与实现分离**——backend 只 expose 简单 API（spawn/sendMessage/terminate），所有复杂逻辑藏在 runner 里。

## 4.12 几个隐性设计判断

### 1. 实现 TeammateExecutor 而不是 PaneBackend

In-process 没有 pane 概念——强行实现 PaneBackend 会要求 stub 一堆 hide/show/setBorderColor 等方法（毫无意义）。

实现 TeammateExecutor 干净——只表达"teammate lifecycle"语义。pane backend 通过适配器变成 TeammateExecutor。

### 2. 文件 mailbox 即使在同进程也用

为了一致性。短期看是多一次文件 IO，长期看是代码路径统一、bug surface 小。

### 3. fire-and-forget startInProcessTeammate

spawn 立刻返回 spawn result，agent loop 在后台跑。teammate 是 actor 不是 RPC——spawn 不该等。

### 4. 显式剥离 messages 字段

`toolUseContext: { ...this.context, messages: [] }`——故意置空。注释解释为什么。这是**防御性编程的清晰版本**——不让无用数据钉住资源。

### 5. terminate 用 mailbox 协议

通过 mailbox 发 shutdown_request 而不是直接调 teammate 函数。原因：
- teammate 可能在处理消息（mid-turn）；
- 应该让 teammate 自己决定何时退出（plan_approval 也是同样模型）；
- 统一异步交互模式。

### 6. kill 用 AbortController

不是 SIGKILL 进程（同进程做不到）——用 AbortController 给所有内部 await 发 abort 信号。这要求 teammate 的所有阻塞操作**正确传 abortSignal**。

### 7. 默认值偏保守

`isActive` 里 `task.abortController?.signal.aborted ?? true`——missing 视为 aborted。这种**默认值偏向 false（更保守的判定）**是关键状态判定的标配。

### 8. isAvailable always true

让 in-process 成为永远的 fallback——detection 失败时至少这个能用。

## 4.13 InProcessBackend 339 行的复杂度来源

虽然只有 339 行，但承担了：

- TeammateExecutor 6 方法实现；
- ToolUseContext 注入与 guard；
- spawn 流程协调（context 创建 + 任务注册 + runner 启动）；
- 与 mailbox 系统对接；
- 与 AbortController 生命周期对接；
- 与 AppState 任务状态对接；
- 与 inProcessRunner 接口对接；
- 错误处理 + logForDebugging。

它的复杂度在于**协调多个独立系统**，而不是自己实现什么。这是个**编排型**模块。

## 4.14 小结

- InProcessBackend 实现 TeammateExecutor 接口，不是 PaneBackend——没有 pane 概念；
- 与 pane backend 3 大差异：进程模型 / 隔离机制 / 生命周期；
- **同进程也走文件 mailbox**——为了代码路径统一；
- ToolUseContext 必须通过 setContext 注入——为了访问 AppState；
- spawn 是 fire-and-forget——teammate 是 actor，不是 RPC；
- 显式剥离 toolUseContext.messages 避免钉住 parent 对话；
- terminate 用 mailbox shutdown 协议、kill 用 AbortController；
- isActive 双重判定 status + signal.aborted；
- isAvailable always true——永远 fallback；
- 339 行的复杂度在编排多个独立系统，不在自己实现什么。

下一篇 → [05 消息系统与 SendMessage](./05-消息系统与SendMessage.md)
