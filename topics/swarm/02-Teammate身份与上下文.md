# 02 Teammate 身份与上下文

> "我是谁、我在哪个团队、我是不是 lead"——3 个核心问题在 swarm 里有 3 种独立答案源（AsyncLocalStorage / dynamicTeamContext / env var）。本篇拆这套优先级链路。

## 2.1 3 种身份信号源

`utils/teammate.ts:1-14` 文件头注释直接说明：

> These helpers identify whether this Claude Code instance is running as a spawned teammate in a swarm. Teammates receive their identity via CLI arguments (--agent-id, --team-name, etc.) which are stored in dynamicTeamContext.
>
> For in-process teammates (running in the same process), AsyncLocalStorage provides isolated context per teammate, preventing concurrent overwrites.
>
> Priority order for identity resolution:
> 1. AsyncLocalStorage (in-process teammates) - via teammateContext.ts
> 2. dynamicTeamContext (tmux teammates via CLI args)

3 种身份信号源 + 默认（lead 没有任何信号）：

| 信号源 | 用于 | 例 |
|--------|------|---|
| **AsyncLocalStorage** | in-process teammate | 同进程跑多个 teammate 时 |
| **dynamicTeamContext** (模块级 var) | tmux/iTerm2 teammate | 通过 CLI flag 启动的独立进程 |
| **环境变量** | 部分功能（如 `CLAUDE_CODE_PLAN_MODE_REQUIRED`） | 历史 fallback |
| **(无信号)** | team-lead 或非 swarm 单 agent 模式 | 默认 |

`utils/teammate.ts:88-92` 的 `getAgentId()` 体现优先级：

```ts
export function getAgentId(): string | undefined {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.agentId   // 1. AsyncLocalStorage
  return dynamicTeamContext?.agentId               // 2. dynamicTeamContext
}
```

这套优先级几乎贯穿所有身份查询函数（`getAgentName` / `getTeamName` / `getTeammateColor` / `isPlanModeRequired`）。

## 2.2 AsyncLocalStorage —— in-process 隔离基石

`utils/teammateContext.ts` 是这整套机制的核心。整个文件 96 行——但概念密度极高。

### TeammateContext 类型

```ts
export type TeammateContext = {
  agentId: string                  // "researcher@my-team"
  agentName: string                // "researcher"
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string          // lead 的 session ID
  isInProcess: true                // 判别符
  abortController: AbortController // 生命周期与 parent 关联
}

const teammateContextStorage = new AsyncLocalStorage<TeammateContext>()
```

`AsyncLocalStorage` 是 Node.js 内置的"基于异步执行栈隔离的存储"——同一个进程里的多个并发异步链可以**各自看到自己的 store**。

### runWithTeammateContext —— 进入隔离作用域

```ts
export function runWithTeammateContext<T>(
  context: TeammateContext,
  fn: () => T,
): T {
  return teammateContextStorage.run(context, fn)
}
```

`AsyncLocalStorage.run(ctx, fn)` 让 `fn` 及其异步派生（`.then`、`await`、`setTimeout` 等）都能通过 `getStore()` 拿到 `ctx`——但**外部**和**其它 run 调用**看不到。

举例：

```ts
// 在主进程跑：
runWithTeammateContext(ctx1, async () => {
  console.log(getTeammateContext()?.agentName)  // "researcher"
  await someAsyncOp()
  console.log(getTeammateContext()?.agentName)  // "researcher" (await 后仍能拿到)
})

runWithTeammateContext(ctx2, async () => {
  console.log(getTeammateContext()?.agentName)  // "coder"
})

console.log(getTeammateContext())                // undefined (在外面)
```

3 个 teammate 在同一个进程里跑——它们的 `cwd` / `agentId` / `teamName` 都通过 AsyncLocalStorage 隔离。

### getTeammateContext / isInProcessTeammate

```ts
export function getTeammateContext(): TeammateContext | undefined {
  return teammateContextStorage.getStore()
}

export function isInProcessTeammate(): boolean {
  return teammateContextStorage.getStore() !== undefined
}
```

**返回 undefined 意味着不在任何 in-process teammate 的执行链上**——可能是主进程（lead）或非 swarm 单 agent。

### createTeammateContext —— 工厂函数

```ts
export function createTeammateContext(config: {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId: string
  abortController: AbortController
}): TeammateContext {
  return {
    ...config,
    isInProcess: true,
  }
}
```

简单工厂——加上 `isInProcess: true` 判别符。

### abortController 设计：故意不链接到 parent

注释（`teammateContext.ts:76-81`）很关键：

> The abortController is passed in by the caller. For in-process teammates, this is typically an independent controller (**not linked to parent**) so teammates continue running when the leader's query is interrupted.

设计选择：**teammate 的 abortController 不绑定到 lead 的 abortController**。lead 按 ESC 中断自己当前 query 时，**teammates 仍继续跑**。

理由：teammate 是独立 actor——不是 sub-agent。lead 中断自己的当前回复不应该影响 teammate 的工作流。要 stop teammate 用专门的 shutdown 协议（[06](./06-协议消息状态机.md)）。

## 2.3 dynamicTeamContext —— tmux teammate 的入口

`utils/teammate.ts:44-81` 是 dynamicTeamContext 部分。

### 数据形态

```ts
let dynamicTeamContext: {
  agentId: string
  agentName: string
  teamName: string
  color?: string
  planModeRequired: boolean
  parentSessionId?: string
} | null = null
```

**模块级 mutable 变量**——只有一个值。这意味着 tmux teammate 模型下，**每个进程只能是一个 teammate**（不像 in-process 可以同进程跑多个）。

### setDynamicTeamContext / clearDynamicTeamContext

```ts
export function setDynamicTeamContext(context: { ... } | null): void {
  dynamicTeamContext = context
}

export function clearDynamicTeamContext(): void {
  dynamicTeamContext = null
}
```

tmux teammate 在启动时（CLI 解析后）调 `setDynamicTeamContext`——把 CLI flag 解析出的身份信息存进去。整个生命周期不变。

注释提"set when joining a team at runtime"——也可以**运行时加入团队**（用户在某个普通 Claude 实例里通过命令加入团队）。这种模式比较少见但支持。

### CLI flag 路径

tmux teammate 通过 CLI flag 接收身份：

```bash
claude --agent-id "researcher@my-team" --team-name "my-team" --agent-name "researcher" ...
```

启动脚本（在 setup.ts 或 init.ts，本篇不展开）解析这些 flag → 调 `setDynamicTeamContext({ agentId, teamName, agentName, ... })` → 后续代码通过 `getAgentId()` 等查询就能拿到。

## 2.4 优先级实现细节

`getAgentId` / `getAgentName` / `getTeamName` 等所有 getter 走同一模板：

```ts
export function getX(): T | undefined {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.x
  return dynamicTeamContext?.x
}
```

但 `getTeamName` 有个**特殊路径**（`teammate.ts:111-118`）：

```ts
export function getTeamName(teamContext?: {
  teamName: string
}): string | undefined {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.teamName
  if (dynamicTeamContext?.teamName) return dynamicTeamContext.teamName
  return teamContext?.teamName  // 第三优先级: 调用者传的 AppState 参数
}
```

接受一个可选 `teamContext` 参数。注释解释：

> Pass teamContext from AppState to support leaders who don't have dynamicTeamContext set.

**Lead 没有 dynamicTeamContext**（因为它不是被 spawn 出来的 teammate），但 lead 知道自己的 team name（存在 AppState.teamContext 里）。所以 `getTeamName(appState.teamContext)` 让 lead 也能查到自己的 team。

这种"接受额外参数作为最后 fallback" 是个轻量的设计 trick——让一个函数同时服务 lead 和 teammate。

### isPlanModeRequired —— 3 个 fallback 层

```ts
export function isPlanModeRequired(): boolean {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return inProcessCtx.planModeRequired
  if (dynamicTeamContext !== null) {
    return dynamicTeamContext.planModeRequired
  }
  return isEnvTruthy(process.env.CLAUDE_CODE_PLAN_MODE_REQUIRED)
}
```

3 层 fallback：

1. AsyncLocalStorage 优先；
2. dynamicTeamContext 次之；
3. 环境变量兜底。

环境变量是**历史 fallback**——为了向后兼容老脚本（启动 Claude 时用 env 而不是 flag）。

## 2.5 isTeammate / isTeamLead —— 角色判定

### isTeammate

```ts
export function isTeammate(): boolean {
  const inProcessCtx = getTeammateContext()
  if (inProcessCtx) return true
  return !!(dynamicTeamContext?.agentId && dynamicTeamContext?.teamName)
}
```

**有 AsyncLocalStorage** → 一定是 in-process teammate → true；
**没 AsyncLocalStorage 但有 dynamicTeamContext** → 必须 agentId 和 teamName 都有 → true；
否则 → false（lead 或独立 Claude 实例）。

为什么 tmux teammate 要"agentId 和 teamName 都有"？因为部分场景可能只设了一个（如未完全启动）——保守判定为 false。

### isTeamLead

```ts
export function isTeamLead(
  teamContext: { leadAgentId: string } | undefined,
): boolean {
  if (!teamContext?.leadAgentId) return false

  const myAgentId = getAgentId()
  const leadAgentId = teamContext.leadAgentId

  // 1. 我的 agentId 等于 lead 的 agentId
  if (myAgentId === leadAgentId) return true

  // 2. 我没设 agentId 但有 team context (向后兼容)
  if (!myAgentId) return true

  return false
}
```

`isTeamLead` **接受 teamContext 参数**——因为 lead 自己的 agentId 信息不在自己的身份里（lead 不是 teammate），而在 AppState.teamContext 里。

注释明确两种情况都算 lead：

```
2. Either:
   - Our CLAUDE_CODE_AGENT_ID matches the leadAgentId, OR
   - We have no CLAUDE_CODE_AGENT_ID set (backwards compat: the original
     session that created the team before agent IDs were standardized)
```

**向后兼容路径**：早期版本 lead 没有 agentId 概念（只是普通 session 创建了团队）。后来加了 agentId 后保持兼容——没设 agentId 但有 team context 仍当作 lead。

## 2.6 hasActiveInProcessTeammates / waitForTeammatesToBecomeIdle —— lifecycle 协调

### hasActiveInProcessTeammates

```ts
export function hasActiveInProcessTeammates(appState: AppState): boolean {
  for (const task of Object.values(appState.tasks)) {
    if (task.type === 'in_process_teammate' && task.status === 'running') {
      return true
    }
  }
  return false
}
```

扫描 AppState 里所有 task，找 type 是 'in_process_teammate' 且 status 是 'running' 的。**用于 headless / print 模式判断"应不应该退出"**——有 teammate 还在跑就等。

### waitForTeammatesToBecomeIdle

```ts
export function waitForTeammatesToBecomeIdle(
  setAppState: (...) => void,
  appState: AppState,
): Promise<void> {
  const workingTaskIds: string[] = []

  for (const [taskId, task] of Object.entries(appState.tasks)) {
    if (task.type === 'in_process_teammate' && task.status === 'running' && !task.isIdle) {
      workingTaskIds.push(taskId)
    }
  }

  if (workingTaskIds.length === 0) return Promise.resolve()

  return new Promise<void>(resolve => {
    let remaining = workingTaskIds.length
    const onIdle = (): void => {
      remaining--
      if (remaining === 0) resolve()
    }

    setAppState(prev => {
      const newTasks = { ...prev.tasks }
      for (const taskId of workingTaskIds) {
        const task = newTasks[taskId]
        if (task && task.type === 'in_process_teammate') {
          if (task.isIdle) {
            // Race condition: 我们记录时它还在跑, 现在它已 idle
            onIdle()
          } else {
            newTasks[taskId] = {
              ...task,
              onIdleCallbacks: [...(task.onIdleCallbacks ?? []), onIdle],
            }
          }
        }
      }
      return { ...prev, tasks: newTasks }
    })
  })
}
```

**等所有 teammate idle**——常见用法："lead 准备 shutdown 前先等 teammate 完成当前 turn"。

注意 race 处理：

```
Check current isIdle state to handle race where teammate became idle
between our initial snapshot and this callback registration
```

从初次扫描到设置 callback 之间的极短窗口，teammate 可能刚好 idle 了。所以再 check 一次 `isIdle`——如果 true，立刻调 `onIdle` 而不是等永远不会来的回调。

这种**初次扫描 + setAppState 内再 check**的 race 防御是 React 状态管理的常用模式。

## 2.7 整体流程：teammate 启动 → 跑 → idle 的身份变化

### 路径 A：in-process teammate

```
1. lead 调 spawnInProcess
                  │
                  ▼
2. spawnInProcess 准备 context:
   const ctx = createTeammateContext({
     agentId: 'researcher@my-team',
     agentName: 'researcher',
     teamName: 'my-team',
     color: 'red',
     planModeRequired: false,
     parentSessionId: leadSessionId,
     abortController: new AbortController(),  // 独立, 不绑 parent
   })
                  │
                  ▼
3. runWithTeammateContext(ctx, async () => {
     // 整个 teammate 的执行链在这个 closure 里
     await runAgent(...)
   })
                  │
                  ▼
4. teammate 内部任何 await 都能 getTeammateContext() 拿到 ctx
   - getAgentId() → 'researcher@my-team'
   - getAgentName() → 'researcher'
   - getTeamName() → 'my-team'
   - isTeammate() → true
   - isInProcessTeammate() → true
                  │
                  ▼
5. teammate 完成 turn → task.isIdle = true
                  │
                  ▼
6. 还能继续 (在 ctx 内可重新跑新 turn)
   ctx 一直存活直到 abortController.abort() 或 进程退出
```

### 路径 B：tmux teammate

```
1. lead 调 spawnMultiAgent (tmux backend)
                  │
                  ▼
2. 启动一个新 claude 进程:
   $ claude --agent-id "researcher@my-team" \
            --team-name "my-team" \
            --agent-name "researcher" \
            --parent-session-id "leader-uuid"
                  │
                  ▼
3. 新进程 init.ts:
   setDynamicTeamContext({
     agentId, teamName, agentName, parentSessionId
   })
                  │
                  ▼
4. teammate 内部任何代码:
   - getAgentId() → 'researcher@my-team'  (从 dynamicTeamContext)
   - getTeammateContext() → undefined  (没有 AsyncLocalStorage)
   - isTeammate() → true (因为 dynamicTeamContext 有 agentId + teamName)
   - isInProcessTeammate() → false  (区分点)
                  │
                  ▼
5. 进程持续跑直到 shutdown_request 或 kill
```

## 2.8 几个隐性设计判断

### 1. AsyncLocalStorage 让 in-process 成为可能

没有 Node 18+ 的 AsyncLocalStorage，同进程跑多个 teammate 几乎不可能——所有 globals（cwd / agentId / sessionEnv）都会互相覆盖。

AsyncLocalStorage 是 Node 实验性能特性的成熟应用——agent 系统应该是它最有价值的用例之一。

### 2. dynamicTeamContext 是单例

tmux teammate 模式下，每个进程只能是一个 teammate。这简化了实现——但限制是：**一个 Node 进程不能同时是两种 backend 的 teammate**。

实际不需要——每个 teammate 进程通过 backend 决定路径，要么是 in-process 共享主进程，要么是独立进程。

### 3. 优先级链：AsyncLocalStorage > dynamicTeamContext > 参数 > env

每个 getter 的优先级实现一致。这种"统一模板"让维护成本低——加新 getter 只需要按这个模板写。

### 4. lead 没有"自己的"身份

Lead 的 agentId 在 `AppState.teamContext.leadAgentId` 里，**不在 dynamicTeamContext 也不在 AsyncLocalStorage**。`isTeamLead(teamContext)` 必须接受外部参数。

为什么这样？因为 lead 在用户启动 Claude Code 时就存在——那时还没有任何 swarm 概念。lead 的身份是后来"创建团队"时附加的属性，不是启动时确立的。

### 5. abortController 故意不链接 parent

teammate 是独立 actor——lead 中断自己当前 query 不该 cascade 到 teammate。要 stop teammate 必须走显式 shutdown 协议。

这和 [agent 06 runAgent](../agent/06-runAgent核心机制.md) 里 sub-agent 的设计相反——sub-agent 的 abortController 绑 parent，parent abort 时 sub-agent 也停。两者代表不同的概念边界。

### 6. backwards compat 路径

`isTeamLead` 的 "no agentId + has team context = lead" 路径是历史兼容。这种**显式标注向后兼容**的代码注释让读者知道为什么有 "看起来很弱的判定"。

### 7. waitForTeammatesToBecomeIdle 的 race 处理

setAppState 内再 check `isIdle`——这种"采样 + 校验"模式在 React 异步状态管理里是必需的。注释明确指出 race 的可能窗口。

## 2.9 小结

- 3 种身份信号源 + lead 默认：AsyncLocalStorage / dynamicTeamContext / env var / (无信号)；
- AsyncLocalStorage 是 in-process teammate 隔离的基石——让同进程跑多个 teammate 不互相污染；
- dynamicTeamContext 是模块级单例，tmux teammate 通过 CLI flag 初始化；
- 优先级链统一：所有 getter 都先 check AsyncLocalStorage、再 dynamicTeamContext、最后 env / 参数；
- **lead 没有自己的 dynamicTeamContext**——lead 是先于 swarm 存在的；
- `isTeamLead` 接受 teamContext 参数让 lead 能查到自己（参数作为 fallback）；
- teammate 的 abortController **故意不链接 parent**——teammate 是独立 actor 不是 sub-agent；
- `waitForTeammatesToBecomeIdle` 用 race-aware 模式等所有 teammate 完成 turn；
- `hasActiveInProcessTeammates` 让 headless / print 模式知道"该不该退出"。

下一篇 → [03 Pane backend (tmux + iTerm2)](./03-Pane-backend-tmux-iterm2.md)
