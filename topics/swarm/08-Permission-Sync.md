# 08 Permission Sync

> Worker teammate 没有本地权限决策——它把请求**通过文件系统**传给 leader，leader 弹窗给用户，response 通过文件传回来。`permissionSync.ts` 928 行解决这条"分布式权限决策"链路。本篇拆请求文件结构、leader/worker 角色协议、leaderPermissionBridge 怎么把请求接入 UI、reconnection 启动时的 context 恢复。

## 8.1 为什么 worker 不本地决策

回顾 [00 总览](./00-总览与代码地图.md)：

> worker 没有本地决策权——通过 leaderPermissionBridge 把请求传回 lead

3 个原因：

### 1. UI 渲染权由 lead 持有

终端（terminal）只有一个——Ink UI 是个有状态渲染器，多个进程不能同时渲染同一终端。**Lead 进程占用 UI**，所有用户交互（包括权限对话框）都得通过 lead。

worker pane 里也有"Claude 在跑"的视觉，但那是 worker 自己的 Ink 实例渲染在它自己的 pane 里——里面没有权限对话框的入口（因为最终决定权在 lead）。

### 2. 用户体验一致性

如果 worker 也能弹对话框——用户会同时被 N 个 pane 里的对话框打断。**集中到 lead 让用户只看一个权限流**。

### 3. Audit / 共享 rules

所有权限决策走 lead 的 `ToolPermissionContext` → 用户加的 "always allow" 规则**自动覆盖所有 teammate**。如果每个 teammate 独立决策，规则不共享。

## 8.2 两套协议路径：文件 vs mailbox

`permissionSync.ts` 实现的是**专门的文件系统协议**——和 [06 协议消息](./06-协议消息状态机.md) 讲的 mailbox 协议**并行**。

| 路径 | 用途 | 数据位置 |
|------|------|---------|
| **Mailbox 协议** (06) | 普通消息 + shutdown/plan_approval 等异步事件 | `~/.claude/teams/{team}/inboxes/*.json` |
| **PermissionSync 协议** (本篇) | 高频权限请求 + 即时响应 | `~/.claude/teams/{team}/permissions/pending/*.json` + `resolved/*.json` |

为什么用两套？

- mailbox 一个 inbox 是个 array，多 reader/writer 时锁竞争激烈；
- 权限请求**每条独立成文件**避免共享锁；
- 权限请求需要 leader 主动 poll pending dir（不能等 attachment surfacer 注入）；
- 这种细粒度高频交互不适合 mailbox 的"消息追加"模型。

文件协议的目录结构：

```
~/.claude/teams/{teamName}/permissions/
├── pending/                          ← worker 写请求
│   ├── perm-1719857234234-abc123.json
│   └── perm-1719857234567-def456.json
├── resolved/                         ← leader 写决议
│   ├── perm-1719857234234-abc123.json
│   └── perm-1719857234567-def456.json
└── .lock                             ← 目录级锁
```

**pending → resolved 流转**：worker 写 pending、leader 读 pending 并响应、resolved 文件供 worker 读取。

## 8.3 SwarmPermissionRequest schema

`permissionSync.ts:49-90`：

```ts
export const SwarmPermissionRequestSchema = lazySchema(() =>
  z.object({
    id: z.string(),
    workerId: z.string(),                  // 发起 worker 的 agentId
    workerName: z.string(),
    workerColor: z.string().optional(),
    teamName: z.string(),

    toolName: z.string(),                  // 要跑的工具 ('Bash' / 'Edit' / ...)
    toolUseId: z.string(),                 // worker context 里的 toolUseId
    description: z.string(),               // 人话描述
    input: z.record(z.string(), z.unknown()),  // 工具参数
    permissionSuggestions: z.array(z.unknown()),  // 建议规则

    status: z.enum(['pending', 'approved', 'rejected']),
    resolvedBy: z.enum(['worker', 'leader']).optional(),
    resolvedAt: z.number().optional(),
    feedback: z.string().optional(),       // 拒绝理由
    updatedInput: z.record(z.string(), z.unknown()).optional(),  // 修改后参数
    permissionUpdates: z.array(z.unknown()).optional(),  // 应用的 "always allow"
    createdAt: z.number(),
  }),
)
```

16 个字段——**同一个 schema 在 pending 和 resolved 阶段都用**。区别在 `status` 字段：

- pending 写入时：`status: 'pending'`、`resolvedBy`/`resolvedAt`/`feedback`/`updatedInput`/`permissionUpdates` 都 undefined；
- resolved 写入时：status 改 `'approved'` 或 `'rejected'`，其它字段填充。

**Read once, write twice**——一个 schema 服务两个阶段，简化代码。

### 关键字段

| 字段 | 用途 |
|------|------|
| `id` | request ID，文件名也用它 |
| `workerId` / `workerName` / `workerColor` | 发起 worker 的身份与 UI 标识 |
| `teamName` | 路由（其实从路径就能推） |
| `toolName` / `toolUseId` / `input` | 工具调用本身 |
| `description` | 弹窗显示给用户的人话描述 |
| `permissionSuggestions` | 建议的 "always allow" 规则列表 |
| `status` | 状态机的核心字段 |
| `resolvedBy` | worker 自己解决了（用户超时）vs leader 解决 |
| `feedback` | 拒绝时的理由 |
| `updatedInput` | leader 可能修改 input（如把 path 改成 absolute） |
| `permissionUpdates` | 用户勾选 "always allow X" 时产生的新规则 |

`updatedInput` 是个有趣的设计——leader 可以在批准时**改写 input**，worker 拿到 resolved 后用新 input 跑。这让 user 能"修正"模型的工具参数（如 typo 修复）。

## 8.4 writePermissionRequest —— worker 写请求

`permissionSync.ts:215-250`：

```ts
export async function writePermissionRequest(
  request: SwarmPermissionRequest,
): Promise<SwarmPermissionRequest> {
  await ensurePermissionDirsAsync(request.teamName)

  const pendingPath = getPendingRequestPath(request.teamName, request.id)
  const lockDir = getPendingDir(request.teamName)
  const lockFilePath = join(lockDir, '.lock')
  await writeFile(lockFilePath, '', 'utf-8')

  let release: (() => Promise<void>) | undefined
  try {
    release = await lockfile.lock(lockFilePath)

    // Write the request file
    await writeFile(pendingPath, jsonStringify(request, null, 2), 'utf-8')

    return request
  } catch (error) {
    throw error
  } finally {
    if (release) await release()
  }
}
```

3 个细节：

### 1. 目录级 lock 而不是文件级 lock

```ts
const lockFilePath = join(lockDir, '.lock')  // pending/.lock
```

锁 **`pending/.lock`** 而不是 `${pendingPath}.lock`——锁整个 pending 目录。

为什么？因为多个 worker 可能同时写不同 request 文件——但 leader **扫描整个 pending 目录**也需要原子性。统一目录级锁让 reader 和 writer 都不会读到部分写入的状态。

代价：写并发降低——多 worker 写不同 request 仍要排队。但权限请求频率不高，影响不大。

### 2. 写之前先 ensure dirs

```ts
await ensurePermissionDirsAsync(request.teamName)
```

`ensurePermissionDirsAsync` 创建 `permissions/` + `pending/` + `resolved/` 三层目录。每次写都 ensure——**幂等**，目录已存在不报错。

### 3. mkfile + 加锁顺序

```ts
await writeFile(lockFilePath, '', 'utf-8')  // 1. 先创建 lock 文件
release = await lockfile.lock(lockFilePath)  // 2. 锁它
```

proper-lockfile 要求锁对象**文件存在**。先 `writeFile(lockFilePath, '')` 创建空文件再加锁。`'wx'` 模式不需要（即使存在也用空内容覆盖一下没关系，因为加锁后内容不重要）。

## 8.5 readPendingPermissions —— leader 扫描

`permissionSync.ts:256-318`（精简）：

```ts
export async function readPendingPermissions(teamName?: string): Promise<SwarmPermissionRequest[]> {
  const team = teamName || getTeamName()
  const pendingDir = getPendingDir(team)

  let release: (() => Promise<void>) | undefined
  try {
    release = await lockfile.lock(lockFilePath)

    const files = await readdir(pendingDir)
    const requests: SwarmPermissionRequest[] = []
    for (const file of files) {
      if (!file.endsWith('.json')) continue
      const content = await readFile(join(pendingDir, file), 'utf-8')
      const parsed = SwarmPermissionRequestSchema().safeParse(jsonParse(content))
      if (parsed.success) requests.push(parsed.data)
    }
    return requests
  } finally {
    if (release) await release()
  }
}
```

读全部 `*.json`、过 schema validate、返回。**也加锁**——避免读到部分写入。

Leader 主循环里定期调这个函数，发现 pending request 就弹 UI 对话框。

## 8.6 resolvePermission —— leader 写决议

`permissionSync.ts:360-450`（精简）：

```ts
export async function resolvePermission(
  requestId: string,
  resolution: PermissionResolution,
  teamName?: string,
): Promise<SwarmPermissionRequest | null> {
  const team = teamName || getTeamName()
  // ... 锁定 ...

  const pendingPath = getPendingRequestPath(team, requestId)
  const resolvedPath = getResolvedRequestPath(team, requestId)

  try {
    // 1. 读 pending request
    const content = await readFile(pendingPath, 'utf-8')
    const request = SwarmPermissionRequestSchema().parse(jsonParse(content))

    // 2. 更新字段
    const resolved: SwarmPermissionRequest = {
      ...request,
      status: resolution.decision === 'approved' ? 'approved' : 'rejected',
      resolvedBy: resolution.resolvedBy,
      resolvedAt: Date.now(),
      feedback: resolution.feedback,
      updatedInput: resolution.updatedInput,
      permissionUpdates: resolution.permissionUpdates as unknown[],
    }

    // 3. 写到 resolved/
    await writeFile(resolvedPath, jsonStringify(resolved, null, 2), 'utf-8')

    // 4. 删除 pending/
    await unlink(pendingPath)

    return resolved
  } finally {
    // 释放锁
  }
}
```

**state 转换的 4 步**：

1. 读 pending；
2. 修改状态字段；
3. 写 resolved；
4. 删 pending（让下次扫描不再处理）。

**关键顺序**：先写 resolved 再删 pending。**反过来不行**——如果先删 pending、写 resolved 失败 → 请求**永久丢失**（pending 不在了、resolved 没成功）。

这是个**两阶段持久化**模式——保证状态转换不会丢失中间状态。

## 8.7 pollForResponse —— worker 等待响应

`permissionSync.ts:544-568`（精简）：

```ts
export async function pollForResponse(
  requestId: string,
  teamName: string,
  signal: AbortSignal,
): Promise<PermissionResponse> {
  while (!signal.aborted) {
    const resolved = await readResolvedPermission(requestId, teamName)
    if (resolved) {
      return {
        decision: resolved.status === 'approved' ? 'approved' : 'rejected',
        feedback: resolved.feedback,
        updatedInput: resolved.updatedInput,
        permissionUpdates: resolved.permissionUpdates as PermissionUpdate[],
      }
    }

    await sleep(POLL_INTERVAL_MS)  // ~500ms
  }
  throw new Error('Aborted')
}
```

worker 写完 pending request 后**阻塞**等待——轮询 resolved/ 目录。500ms 间隔（与 [07 In-process Runner](./07-In-process-Runner.md) 的 mailbox poll 同步）。

resolved 出现 → 解析、返回。被 abort → 抛错。

这是**完全同步等待**——worker 在权限决策期间**完全阻塞**，不接受其它请求。这简化了一致性（不会有"等待中又来新请求"的复杂情况），代价是 worker 不能并发跑工具。

## 8.8 角色判定：isTeamLeader / isSwarmWorker

`permissionSync.ts:581-606`（精简）：

```ts
export function isTeamLeader(teamName?: string): boolean {
  // 通过 AppState.teamContext 判定
  // 不在 swarm 里 → false
  // 不是 lead → false
  // 是 lead → true
}

export function isSwarmWorker(): boolean {
  return isTeammate() && !isTeamLead(...)
}
```

3 种角色：
- **leader**：管理团队、UI 持有者、决策权限；
- **worker**：teammate 不是 lead；
- **standalone**：不在 swarm 里。

`isSwarmWorker()` = `isTeammate() && !isTeamLead()`——结合 [02 Teammate 身份](./02-Teammate身份与上下文.md) 的判定。

## 8.9 sendPermissionRequestViaMailbox —— 双路径协议

`permissionSync.ts:676+`（推断）：

```ts
export async function sendPermissionRequestViaMailbox(
  request: SwarmPermissionRequest,
  leaderName: string,
): Promise<void> {
  // 同时走 mailbox 通知 leader
  const message = createPermissionRequestMessage({...})
  await writeToMailbox(leaderName, {
    text: JSON.stringify(message),
    from: request.workerName,
    timestamp: ...,
  }, request.teamName)
}
```

注意：**permissionSync 同时写 pending dir 和 mailbox**。两条路径冗余但有用：

- **pending dir** 是"权威 source of truth"——leader poll 它做决策；
- **mailbox** 是**通知**——让 leader 不需要高频 poll 也能立刻知道有新请求。

实际上 leader 看到 mailbox 消息后调 `readPendingPermissions` 拿权威数据。

mailbox 起 "doorbell" 作用——告诉 leader "去看 pending dir"。pending dir 是 "letter box"——存实际请求。

这种 **doorbell + letterbox 模式**让通知低延迟、状态有权威源。

## 8.10 leaderPermissionBridge —— REPL 接入

`leaderPermissionBridge.ts` 全文 54 行——非常简单的桥：

```ts
let registeredSetter: SetToolUseConfirmQueueFn | null = null
let registeredPermissionContextSetter: SetToolPermissionContextFn | null = null

export function registerLeaderToolUseConfirmQueue(setter): void {
  registeredSetter = setter
}

export function getLeaderToolUseConfirmQueue(): SetToolUseConfirmQueueFn | null {
  return registeredSetter
}

export function unregisterLeaderToolUseConfirmQueue(): void {
  registeredSetter = null
}
```

模块级单例——REPL 启动时 `registerLeaderToolUseConfirmQueue(setter)`，runner 从模块拿。

注释（`leaderPermissionBridge.ts:3-10`）：

> Module-level bridge that allows the REPL to register its setToolUseConfirmQueue and setToolPermissionContext functions for in-process teammates to use.
>
> When an in-process teammate requests permissions, it uses the standard ToolUseConfirm dialog rather than the worker permission badge. This bridge makes the REPL's queue setter and permission context setter accessible from non-React code in the in-process runner.

**bridge 解决"非 React 代码访问 React state setter"** 的问题：

- REPL 是 React 组件——`setToolUseConfirmQueue` 是 useState 的 setter；
- in-process teammate 的 runner 是普通 async function——不在 React 树里；
- 需要让 runner 调 REPL 的 setter；
- **不能** 把 setter 通过 props 传——runner 不在组件树；
- 解法：模块级单例——REPL 启动时 register、runner 用时 get。

这是个**经典的 React + non-React 代码桥接**模式。代价：模块级全局状态、生命周期管理复杂（注册 / 注销）。

### `unregister` 防内存泄漏

```ts
export function unregisterLeaderToolUseConfirmQueue(): void {
  registeredSetter = null
}
```

REPL 卸载时必须 unregister——否则 setter 引用 React state 闭包，state 不能 GC。

## 8.11 reconnection —— 启动时上下文恢复

`reconnection.ts:23-65` 的 `computeInitialTeamContext`：

```ts
export function computeInitialTeamContext(): AppState['teamContext'] | undefined {
  const context = getDynamicTeamContext()
  if (!context?.teamName || !context?.agentName) return undefined

  const { teamName, agentId, agentName } = context

  // Read team file
  const teamFile = readTeamFile(teamName)
  if (!teamFile) return undefined

  const teamFilePath = getTeamFilePath(teamName)
  const isLeader = !agentId  // 没 agentId 说明是 lead

  return {
    teamName,
    teamFilePath,
    leadAgentId: teamFile.leadAgentId,
    selfAgentId: agentId,
    selfAgentName: agentName,
    isLeader,
    teammates: {},
  }
}
```

启动时调用——**before first render**——计算 `AppState.teamContext` 初值。注释明确：

> This is called synchronously in main.tsx to compute the teamContext BEFORE the first render, eliminating the need for useEffect workarounds.

`useEffect` 是 React 的 mount 后才跑的——如果 team context 在 mount 后才初始化，第一帧 UI 会"闪烁"（先渲染空 → 然后有 context 重渲染）。**同步初始化**让第一帧就有正确状态。

### initializeTeammateContextFromSession

`reconnection.ts:75-114`：

```ts
export function initializeTeammateContextFromSession(
  setAppState: ...,
  teamName: string,
  agentName: string,
): void {
  const teamFile = readTeamFile(teamName)
  if (!teamFile) return

  const member = teamFile.members.find(m => m.name === agentName)
  const agentId = member?.agentId

  setAppState(prev => ({
    ...prev,
    teamContext: {
      teamName,
      teamFilePath: getTeamFilePath(teamName),
      leadAgentId: teamFile.leadAgentId,
      selfAgentId: agentId,
      selfAgentName: agentName,
      isLeader: false,
      teammates: {},
    },
  }))
}
```

**resume session 时调用**——从 transcript 里读到的 teamName/agentName，重建 teamContext。

这让 swarm session **能 resume**——Claude Code 重启后能"接着"做 swarm 工作。否则一旦关闭 Claude，所有 teammate 状态丢失。

注意 `isLeader: false`——resume 路径**只用于 teammate**。lead 的 resume 走另一条路径（通过 dynamicTeamContext + computeInitialTeamContext）。

## 8.12 cleanupOldResolutions —— 过期清理

`permissionSync.ts:452+`（推断）：

```ts
export async function cleanupOldResolutions(teamName: string): Promise<void> {
  const resolvedDir = getResolvedDir(teamName)
  const files = await readdir(resolvedDir)

  for (const file of files) {
    const stat = await stat(join(resolvedDir, file))
    const age = Date.now() - stat.mtimeMs
    if (age > MAX_AGE) {
      await unlink(join(resolvedDir, file))
    }
  }
}
```

resolved/ 目录里的文件**不会自动删除**——worker 读完后可能不立刻删（防 race）。需要定期清理过期的。

为什么不立刻删？因为 worker 读完后可能崩溃——再启动时还想读这个 response。保留一段时间让"晚到的 worker reader"能拿到。

`cleanupOldResolutions` 在 leader 启动时调一次或定期调——清理几天前的 resolved 文件。

## 8.13 worker self-resolve —— 超时机制

`resolvedBy: z.enum(['worker', 'leader'])` 暗示 worker 也能 resolve 自己的请求。

什么时候？**leader 长时间不响应** —— worker 等了很久没等到，需要自我决议（默认拒绝或允许，根据策略）。

```ts
// 推断的 pollForResponse 超时逻辑
while (!signal.aborted) {
  if (Date.now() - request.createdAt > LEADER_TIMEOUT_MS) {
    // self-resolve
    await selfResolve(requestId, { decision: 'rejected', resolvedBy: 'worker' })
    return { decision: 'rejected', feedback: 'Leader timeout' }
  }
  // ... poll resolved ...
}
```

**leader 失联保护**——worker 不会无限期阻塞。

## 8.14 与 [bashtool] / [agent] 专题串联

回顾 [bashtool 08 执行层](../bashtool/08-执行层与环境.md) 提到：

> worker teammate 跑 Bash 时不能本地决策权限——通过 leaderPermissionBridge 把请求传回 lead 弹窗

具体路径：

```
worker teammate 的模型生成 tool_use: Bash(command="...")
                  │
                  ▼
worker 的 toolUseContext.canUseTool 被调用
                  │
                  ▼
canUseTool = createInProcessCanUseTool (inProcessRunner)
  → 检查 allowedTools 白名单
  → forward 给 leader:
    1. createPermissionRequest({toolName, input, ...})
    2. writePermissionRequest(req)  ← 写 pending/req.json
    3. sendPermissionRequestViaMailbox  ← doorbell
    4. pollForResponse(reqId)  ← 等 resolved/req.json
                  │
                  ▼
leader 的 attachment surfacer 注入 mailbox 消息
                  │
                  ▼
leader 模型看到 permission_request notification
                  │
                  ▼
leader 的 REPL 通过 leaderPermissionBridge 弹 ToolUseConfirm
                  │
                  ▼
用户点 Allow / Deny / Always
                  │
                  ▼
resolvePermission(reqId, { decision: ..., updatedInput, permissionUpdates })
  → 删 pending/req.json
  → 写 resolved/req.json
                  │
                  ▼
worker 的 pollForResponse 读到 resolved → 返回 decision
                  │
                  ▼
worker 的 canUseTool 返回 { behavior: 'allow', updated_input }
                  │
                  ▼
worker 真正执行 Bash 工具
```

**12 步**完成一次 worker 工具权限决策。看起来复杂，但每步都简单——文件 IO + 轮询 + UI 弹窗。

## 8.15 几个隐性设计判断

### 1. 文件协议而不是消息协议

权限请求频率高、且需要 leader poll——用专门的文件协议而不是塞进 mailbox。每条请求独立文件避免 inbox 锁竞争。

### 2. doorbell + letterbox 模式

mailbox 是 doorbell（低延迟通知），pending dir 是 letterbox（权威数据）。两条路径冗余但解决不同问题。

### 3. 一个 schema 服务两阶段

`SwarmPermissionRequest` 在 pending 和 resolved 阶段共用——status 字段区分。简化代码。

### 4. resolved 先写再删 pending

两阶段持久化——保证状态转换不丢中间状态。**写新状态 → 删旧状态**是经典模式。

### 5. 目录级锁而不是文件级

让 leader scan 整个 pending 时不需要锁每个文件。代价：写并发降低。

### 6. updatedInput 让 leader 改写工具参数

不只是 allow/deny——leader 能**修改 input**（如 typo 修复、path 规范化）。这是个**强能力**——用户能"修正"模型的工具调用而不需要让模型重做。

### 7. permissionUpdates 携带 "always allow" 规则

用户勾选 "always allow X" 时产生新规则——通过 resolved 文件**回传给 worker**，worker 自己也更新自己的本地 permission context。一致性保证。

### 8. leaderPermissionBridge 模块级单例

桥接 React 和非 React 代码——模块级全局状态 + register/unregister 生命周期管理。

### 9. computeInitialTeamContext 同步计算

启动时同步算出 teamContext，避免 useEffect 导致第一帧闪烁。**性能 + UX 优化**。

### 10. resolved 文件保留一段时间再清理

让 worker 崩溃重启时仍能读到 response。cleanup 通过过期时间机制，不是立刻删除。

### 11. worker self-resolve 防 leader 失联

worker 不能无限期阻塞——leader 失联时 worker 用默认决议（rejected）自救。

### 12. 双重协议（mailbox + 文件）的一致性

worker 写 pending dir + 同时发 mailbox 通知。leader 收到 mailbox 后再读 pending dir（不直接信任 mailbox 的内容）。这种**"用 letterbox 做权威、doorbell 做通知"**的模式让两条路径不冲突。

## 8.16 928 行的复杂度来源

permissionSync.ts 928 行涵盖：

- SwarmPermissionRequest schema 与类型
- 路径生成 helper（pending / resolved / lock）
- writePermissionRequest 加锁写入
- readPendingPermissions 加锁读取扫描
- resolvePermission 状态转换（写 resolved + 删 pending）
- pollForResponse 轮询等待
- cleanupOldResolutions 过期清理
- 角色判定 helper（isTeamLeader / isSwarmWorker / getLeaderName）
- sendPermissionRequestViaMailbox doorbell
- sandbox permission 平行实现
- removeWorkerResponse / deleteResolvedPermission cleanup
- 各种错误处理 + logForDebugging

每一项都不算特别复杂，但加起来 928 行——**协调多个独立子系统 + 持久化 + 锁 + 通知**就是这个量级。

## 8.17 小结

- worker 没本地权限决策——3 个原因：UI 渲染权 / 一致体验 / audit & 共享 rules；
- 两套协议路径：mailbox（通知）+ permissionSync 文件协议（权威）；
- `~/.claude/teams/{team}/permissions/{pending,resolved}/*.json`；
- SwarmPermissionRequest schema 16 字段，pending 和 resolved 共用 schema；
- 目录级锁让 scan 和 write 互斥；
- resolvePermission 两阶段：写 resolved → 删 pending（顺序关键）；
- pollForResponse 500ms 轮询 + abort 支持；
- doorbell（mailbox）+ letterbox（pending dir）双协议设计；
- leaderPermissionBridge 模块级单例桥接 React 和非 React 代码；
- computeInitialTeamContext 同步计算避免首帧闪烁；
- initializeTeammateContextFromSession 支持 resume；
- worker 能 self-resolve 防 leader 失联；
- updatedInput 让 leader 改写工具参数（强能力）；
- permissionUpdates 把"always allow"规则同步回 worker。

下一篇 → [09 spawnMultiAgent 批量派发](./09-spawnMultiAgent批量派发.md)
