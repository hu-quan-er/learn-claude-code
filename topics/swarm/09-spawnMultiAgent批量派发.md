# 09 spawnMultiAgent 批量派发

> `tools/shared/spawnMultiAgent.ts` 1093 行——从 lead 一次性派多个 teammate 时的协调逻辑。本篇拆 3 种 spawn 路径（split-pane / separate-window / in-process）、CLI flag 继承、命名冲突解决、layout 顺序、失败处理。

## 9.1 spawnMultiAgent 的位置

回顾 [00 总览](./00-总览与代码地图.md)：lead 调 spawnMultiAgent 内部对每个 teammate 调一次 AgentTool。文件头注释：

> Shared spawn module for teammate creation. Extracted from TeammateTool to allow reuse by AgentTool.

`spawnMultiAgent.ts` 被两个工具用：

- **TeammateTool**（已废弃但还在）：直接 spawn 单个 teammate；
- **AgentTool**：当 `team_name` 参数被提供时，把 sub-agent 当 teammate spawn。

入口 `spawnTeammate`（line 1088）：

```ts
export async function spawnTeammate(
  config: SpawnTeammateConfig,
  context: ToolUseContext,
): Promise<{ data: SpawnOutput }> {
  return handleSpawn(config, context)
}
```

**单 teammate 入口**——一次调用 spawn 一个。批量派发由调用方循环调用实现（typically `Promise.all` 并发）。

## 9.2 handleSpawn —— 3 路分发

`spawnMultiAgent.ts:1040-1078`：

```ts
async function handleSpawn(
  input: SpawnInput,
  context: ToolUseContext,
): Promise<{ data: SpawnOutput }> {
  // 1. 检查 in-process feature flag
  if (isInProcessEnabled()) {
    return handleSpawnInProcess(input, context)
  }

  // 2. 探测 pane backend 是否可用
  try {
    await detectAndGetBackend()
  } catch (error) {
    // auto mode 自动 fallback 到 in-process
    if (getTeammateModeFromSnapshot() !== 'auto') {
      throw error  // 显式 'tmux' mode 不 fallback
    }
    markInProcessFallback()
    return handleSpawnInProcess(input, context)
  }

  // 3. backend 可用 → 选 split-pane 或 separate-window
  const useSplitPane = input.use_splitpane !== false
  if (useSplitPane) {
    return handleSpawnSplitPane(input, context)
  }
  return handleSpawnSeparateWindow(input, context)
}
```

3 个 spawn 路径：

| 路径 | 函数 | 用途 |
|------|------|------|
| **in-process** | `handleSpawnInProcess` (line 840) | 同进程 AsyncLocalStorage |
| **split-pane** | `handleSpawnSplitPane` (line 305) | tmux/iTerm2 split 同 window |
| **separate-window** | `handleSpawnSeparateWindow` (line 545) | 独立 window（少见） |

**3 条路径的复杂度不同**：
- in-process 305 行（840-1040 中部分）：调用 spawnInProcessTeammate + startInProcessTeammate；
- split-pane 240 行（305-545）：创建 pane + 写 CLI flag + 启动 claude 进程；
- separate-window 215 行（545-760）：tmux new-window + 启动 claude 进程；
- 共享 helper 占其余空间。

## 9.3 backend detection 与 fallback 策略

```ts
try {
  await detectAndGetBackend()
} catch (error) {
  if (getTeammateModeFromSnapshot() !== 'auto') {
    throw error  // 显式 mode 不 fallback
  }
  markInProcessFallback()
  return handleSpawnInProcess(input, context)
}
```

3 种模式行为：

| `teammateMode` | backend 不可用时 |
|----------------|------------------|
| `'auto'` | 自动 fallback 到 in-process（无感降级） |
| `'tmux'` | 抛 error（用户显式选了 tmux 就别静默换） |
| `'in-process'` | 不走 detectAndGetBackend（已经知道要 in-process） |

`markInProcessFallback()` 记录"本次会话 fallback 了"——影响 UI banner（防止显示"tmux attach claude-swarm-PID" 提示但实际没 tmux session）。

**显式 vs 自动**——尊重用户配置：
- 用户选 'auto' → 系统决定；
- 用户选 'tmux' → 必须 tmux，不可用就报错；
- 用户选 'in-process' → 永远 in-process。

## 9.4 SpawnTeammateConfig

`spawnMultiAgent.ts:122-138`（推断）：

```ts
export type SpawnTeammateConfig = {
  name: string                       // teammate 名字
  prompt: string                     // 初始 prompt
  cwd: string                        // 工作目录
  teamName?: string
  agentType?: string                 // sub-agent type
  model?: string                     // 模型覆盖
  color?: string                     // UI 颜色
  planModeRequired?: boolean
  worktreePath?: string              // git worktree
  use_splitpane?: boolean            // split vs separate window
  permissionMode?: PermissionMode
  systemPrompt?: string
  systemPromptMode?: 'default' | 'replace' | 'append'
  allowedTools?: string[]
  allowPermissionPrompts?: boolean
  description?: string
  invokingRequestId?: string
}
```

20 字段——和 InProcessRunnerConfig 类似但更宽（既能 in-process 又能 pane）。

## 9.5 buildInheritedCliFlags —— CLI flag 继承

`spawnMultiAgent.ts:208-260`：

```ts
function buildInheritedCliFlags(options?: {
  planModeRequired?: boolean
  permissionMode?: PermissionMode
}): string {
  const flags: string[] = []
  const { planModeRequired, permissionMode } = options || {}

  // 1. permission mode 继承 (plan mode 优先于 bypass)
  if (planModeRequired) {
    // 不继承 bypass mode (安全)
  } else if (permissionMode === 'bypassPermissions' || getSessionBypassPermissionsMode()) {
    flags.push('--dangerously-skip-permissions')
  } else if (permissionMode === 'acceptEdits') {
    flags.push('--permission-mode acceptEdits')
  } else if (permissionMode === 'auto') {
    flags.push('--permission-mode auto')
  }

  // 2. model override
  const modelOverride = getMainLoopModelOverride()
  if (modelOverride) {
    flags.push(`--model ${quote([modelOverride])}`)
  }

  // 3. settings path
  const settingsPath = getFlagSettingsPath()
  if (settingsPath) {
    flags.push(`--settings ${quote([settingsPath])}`)
  }

  // 4. inline plugins
  const inlinePlugins = getInlinePlugins()
  for (const pluginDir of inlinePlugins) {
    flags.push(`--plugin-dir ${quote([pluginDir])}`)
  }

  // 5. chrome flag
  const chromeFlagOverride = getChromeFlagOverride()
  if (chromeFlagOverride === true) flags.push('--chrome')
  else if (chromeFlagOverride === false) flags.push('--no-chrome')

  return flags.join(' ')
}
```

5 类 flag 继承：

| flag | 来源 | 说明 |
|------|------|------|
| `--dangerously-skip-permissions` | session 或 permissionMode | bypass 模式 |
| `--permission-mode <X>` | permissionMode | acceptEdits / auto |
| `--model <X>` | CLI override | 强制使用特定模型 |
| `--settings <path>` | CLI flag | 自定义 settings 文件 |
| `--plugin-dir <path>` | inline plugins | 每个 plugin 一个 flag |
| `--chrome` / `--no-chrome` | CLI flag | UI mode |

### plan mode 优先于 bypass

```ts
if (planModeRequired) {
  // 不继承 bypass mode
}
```

**安全优先**——如果 teammate 需要 plan mode（先 plan 再实施），不继承 bypass。注释：

> Plan mode takes precedence over bypass permissions for safety

bypass 让 teammate 跳过所有权限检查——但如果 teammate 应该走 plan 流程（计划要被审批），跳过权限就破坏了计划保障。两者冲突时 plan 赢。

### --permission-mode auto 双重 gate

```ts
} else if (permissionMode === 'auto') {
  // Teammates inherit auto mode so the classifier auto-approves their tool
  // calls too. The teammate's own startup (permissionSetup.ts) handles
  // GrowthBook gate checks and setAutoModeActive(true) independently.
  flags.push('--permission-mode auto')
}
```

注释里说：lead 是 auto mode 时 teammate 也用 auto——让 classifier 同样自动放行 teammate 的工具调用。但 teammate 启动时**自己**再走一遍 GrowthBook gate 检查（防止 classifier 在某些 teammate 进程意外可用而另一些不可用）。

## 9.6 generateUniqueTeammateName —— 命名冲突解决

`spawnMultiAgent.ts:267-303`：

```ts
export async function generateUniqueTeammateName(
  baseName: string,
  teamName: string | undefined,
): Promise<string> {
  if (!teamName) return baseName

  const teamFile = await readTeamFileAsync(teamName)
  if (!teamFile) return baseName

  // 找冲突
  const existingNames = new Set(teamFile.members.map(m => m.name))

  if (!existingNames.has(baseName)) {
    return baseName  // 没冲突
  }

  // 冲突 → 加数字后缀: tester-2, tester-3, ...
  for (let i = 2; i < 100; i++) {
    const candidate = `${baseName}-${i}`
    if (!existingNames.has(candidate)) return candidate
  }

  throw new Error(`Could not generate unique name for ${baseName}`)
}
```

**自动加后缀避免重名**——`tester` 已存在 → 试 `tester-2` / `tester-3` / ... 直到 99。

为什么不让 lead 提前查重？因为多个 spawn 可能**并发**——lead 同时派 3 个 `tester`，每个都得自动加后缀避免冲突。**spawn 函数自己确保唯一**。

注意 `i < 100`——硬上限，超过抛 error。100 个同名 teammate 基本不可能（一个团队 5-10 个 teammate 是常见上限）。

## 9.7 handleSpawnInProcess —— 同进程派发

`spawnMultiAgent.ts:840-1040`（200 行）核心流程：

```ts
async function handleSpawnInProcess(input, context): Promise<{ data: SpawnOutput }> {
  // 1. 确定 teamName / agentName
  const teamName = input.teamName || getTeamName() || generateTeamSlug()
  const baseName = input.name || `agent-${Date.now()}`
  const agentName = await generateUniqueTeammateName(baseName, teamName)

  // 2. 解析模型
  const leaderModel = getLeaderModel()
  const model = resolveTeammateModel(input.model ?? input.agentDefinition?.model, leaderModel)

  // 3. 写 team file 加新 member
  const teamFile = await readTeamFileAsync(teamName)
  if (teamFile) {
    teamFile.members.push({
      agentId: formatAgentId(agentName, teamName),
      name: agentName,
      agentType: input.agentType,
      model,
      joinedAt: Date.now(),
      tmuxPaneId: '',  // in-process 没 pane
      cwd: input.cwd ?? getCwd(),
      subscriptions: [],
      backendType: 'in-process',
      isActive: true,
    })
    await writeTeamFileAsync(teamName, teamFile)
  }

  // 4. spawnInProcessTeammate (创建 context + 注册 task)
  const spawnResult = await spawnInProcessTeammate({
    name: agentName,
    teamName,
    prompt: input.prompt,
    color: input.color,
    planModeRequired: input.planModeRequired ?? false,
  }, context)

  // 5. startInProcessTeammate (fire-and-forget runner)
  if (spawnResult.success) {
    startInProcessTeammate({
      identity: { ... },
      taskId: spawnResult.taskId,
      prompt: input.prompt,
      teammateContext: spawnResult.teammateContext,
      toolUseContext: { ...context, messages: [] },
      abortController: spawnResult.abortController,
      model,
      systemPrompt: input.systemPrompt,
      systemPromptMode: input.systemPromptMode,
      allowedTools: input.permissions,
      // ...
    })
  }

  // 6. 返回 spawn 结果
  return {
    data: {
      success: spawnResult.success,
      agentId: spawnResult.agentId,
      teamName,
      backendType: 'in-process',
      taskId: spawnResult.taskId,
      abortController: spawnResult.abortController,
    },
  }
}
```

6 步：team file 更新 → spawn context → start runner → 返回 spawn 结果。

注意 step 3 **写 team file**：team file 是单一权威源，所有 spawn 都更新它（[01 激活机制](./01-激活机制与团队配置.md) 提过 TeamCreate 创建 teamFile，spawn 加 members）。

## 9.8 handleSpawnSplitPane —— pane 派发

`spawnMultiAgent.ts:305-543`（240 行）核心流程（简化）：

```ts
async function handleSpawnSplitPane(input, context): Promise<{ data: SpawnOutput }> {
  // 1. 确定 teamName / agentName / model（同 in-process）
  // ...

  // 2. 分配 color
  const color = input.color ?? assignTeammateColor(...)

  // 3. 创建 pane
  const { paneId, isFirstTeammate } = await createTeammatePaneInSwarmView(
    agentName,
    color,
  )

  // 4. 构造 CLI 命令
  const teammateCommand = getTeammateCommand()  // claude binary 路径
  const inheritedFlags = buildInheritedCliFlags({
    planModeRequired: input.planModeRequired,
    permissionMode: input.permissionMode,
  })
  const cwdFlag = `--cwd ${quote([input.cwd ?? getCwd()])}`
  const teammateFlags = [
    `--team-name ${quote([teamName])}`,
    `--agent-id ${quote([formatAgentId(agentName, teamName)])}`,
    `--agent-name ${quote([agentName])}`,
    inheritedFlags,
    cwdFlag,
    // ... 其它 flag
  ].filter(Boolean).join(' ')

  const fullCommand = `${teammateCommand} ${teammateFlags}`

  // 5. 在 pane 里运行命令
  await sendCommandToPane(paneId, fullCommand)

  // 6. 设置 pane 样式（border color / title）
  await setPaneBorderColor(paneId, color)
  await setPaneTitle(paneId, agentName, color)
  await enablePaneBorderStatus()

  // 7. 写 team file 加 member（带 paneId / backendType）
  // ...

  // 8. 注册 out-of-process task（让 UI 能跟踪）
  registerOutOfProcessTeammateTask({
    agentId,
    agentName,
    paneId,
    backendType: detectedBackendType,
    ...
  })

  // 9. 等 shell 启动（PANE_SHELL_READY_DELAY_MS）
  await waitForPaneShellReady()

  return { data: { success: true, agentId, paneId, ... } }
}
```

9 步——比 in-process 多了 pane 创建、CLI flag 构造、pane 装饰、out-of-process task 注册等。

## 9.9 getTeammateCommand —— 决定怎么启动 teammate 进程

`spawnMultiAgent.ts:193-206`：

```ts
function getTeammateCommand(): string {
  // 1. 优先用 env var 覆盖（测试 / 自定义环境）
  const envCommand = process.env[TEAMMATE_COMMAND_ENV_VAR]  // CLAUDE_CODE_TEAMMATE_COMMAND
  if (envCommand) {
    return envCommand
  }

  // 2. 在 bundle 模式下用 process.execPath（自己重新跑自己）
  if (isInBundledMode()) {
    return process.execPath
  }

  // 3. 否则用 `claude`（系统 PATH 中查找）
  return 'claude'
}
```

3 个 fallback：

| 来源 | 用途 |
|------|------|
| `CLAUDE_CODE_TEAMMATE_COMMAND` env | 测试 / 自定义启动命令 |
| `process.execPath` | bundle 模式（self-contained binary） |
| `'claude'` | 系统 PATH 中的 claude 命令 |

`isInBundledMode()` 判断当前是不是从打包的 binary 跑——是的话**重新调用自己**（确保 teammate 用同样的 binary，避免 PATH 中可能存在的其它 claude 版本）。

这是个**重要的一致性保证**——lead 是新版 claude、teammate 是旧版 claude 会出兼容问题。bundle 模式下强制 teammate 用同 binary。

## 9.10 CLI flag 列表

每个 teammate 启动时收到这些 CLI flag：

```
claude
  --team-name <teamName>            ← 必需
  --agent-id <agentName@teamName>    ← 必需
  --agent-name <agentName>            ← 必需
  --cwd <path>                       ← 必需
  --parent-session-id <id>           ← 让 teammate 知道自己的 parent
  --model <model>                    ← 可选
  --settings <path>                  ← 可选 (CLI inherit)
  --plugin-dir <path>                ← 可选 (CLI inherit) ×N
  --dangerously-skip-permissions     ← 可选 (CLI inherit)
  --permission-mode auto             ← 可选
  --chrome / --no-chrome             ← 可选
```

5-12 个 flag 让 teammate 进程完全知道自己的身份与配置。

teammate 启动后 init.ts 解析这些 flag → 调用 `setDynamicTeamContext({...})`（[02 Teammate 身份](./02-Teammate身份与上下文.md) 讲过）。

## 9.11 handleSpawnSeparateWindow —— 独立窗口（少见）

`spawnMultiAgent.ts:545-760`（215 行）——和 split-pane 差不多但用 tmux new-window 而不是 split-window。

什么时候用？

- 用户显式要求；
- pane 数量过多（如 10+ teammate），split 已经不实用；
- 调试用——单独窗口便于切换。

实际 99% 场景用 split-pane。这条路径主要是为完整性提供。

## 9.12 registerOutOfProcessTeammateTask —— UI 跟踪

`spawnMultiAgent.ts:760-840`（80 行）：

```ts
function registerOutOfProcessTeammateTask(config: {
  agentId: string
  agentName: string
  paneId: string
  backendType: BackendType
  // ...
}, context: ToolUseContext): void {
  const task: TaskState = {
    type: 'out_of_process_teammate',
    id: generateTaskId(),
    status: 'running',
    identity: { ... },
    paneId: config.paneId,
    backendType: config.backendType,
    // ...
  }

  context.setAppState(prev => ({
    ...prev,
    tasks: { ...prev.tasks, [task.id]: task },
  }))

  registerTask(task.id, ...)
}
```

每个 spawn 出的 pane teammate 在 lead 的 AppState 里**注册一个 `out_of_process_teammate` task**——让 UI 能：

- 显示 teammate 状态（running / idle / shutdown）；
- 让用户点 hide/show/kill；
- 跟踪 spinner / progress。

注意 task 类型有两种：`in_process_teammate` 和 `out_of_process_teammate`。这两类 task 在 [tasks/InProcessTeammateTask/](../../../claude-code-sourcemap/restored-src/src/tasks/InProcessTeammateTask/) 和 LocalAgentTask 里分别处理。

## 9.13 hasSession / ensureSession —— tmux session 管理

`spawnMultiAgent.ts:159-191`：

```ts
async function hasSession(sessionName: string): Promise<boolean> {
  const result = await execFileNoThrow(TMUX_COMMAND, ['has-session', '-t', sessionName])
  return result.code === 0
}

async function ensureSession(sessionName: string): Promise<void> {
  if (await hasSession(sessionName)) return

  // Create new session
  await execFileNoThrow(TMUX_COMMAND, [
    'new-session',
    '-d',                      // detached
    '-s', sessionName,
    '-n', SWARM_VIEW_WINDOW_NAME,
  ])
}
```

每次 spawn 前**确保 session 存在**——`hasSession` 检查、`ensureSession` 不存在就创建。

`-d` flag 让 session detached（不 attach）——用户没在 tmux 里跑 Claude 时不会突然切到 tmux 里。

如果用户在 tmux 里跑——使用现有 session，不创建新的。

## 9.14 PANE_SHELL_READY_DELAY_MS —— 等 shell 启动

```ts
// 推断
const PANE_SHELL_READY_DELAY_MS = 500
await waitForPaneShellReady()  // sleep(500ms)
```

创建 pane 后**等 500ms 再发命令**——给 shell 启动 + 加载 rc 文件的时间。如果不等：
- send-keys 可能在 shell 还没 ready 时发送 → 命令丢失；
- 或者 shell 加载 rc 文件输出 → 干扰 claude 命令显示。

500ms 是个**经验值**——大部分 shell 启动 < 100ms，但慢一点的环境（NFS 加载、网络 home dir、复杂 .bashrc）可能要 200-300ms。500ms 保守。

## 9.15 部分失败处理

当 lead 一次性派 3 个 teammate 时（外层 `Promise.all`），有可能：

```
teammate-1: spawn 成功
teammate-2: spawn 失败 (pane 创建失败 / shell 启动失败)
teammate-3: spawn 成功
```

`spawnTeammate` 对每个独立调用——失败时返回 `{ success: false, error }`，不抛错。

调用方（如 spawnMultiAgent caller）拿到 3 个 result 后处理：

```ts
const results = await Promise.all(
  teammates.map(t => spawnTeammate(t, context)),
)

const failed = results.filter(r => !r.data.success)
if (failed.length > 0) {
  // 部分失败处理: ...
}
```

**没有"全部成功才算成功"** 的约束——部分失败是常态，lead 自己处理（如重试失败的、或者通知用户）。

### 失败时的清理

如果 spawn 成功但 teammate 启动后失败（shell ready 失败、claude 启动失败）——已创建的 pane 留下来吗？

`waitForPaneShellReady` 之后**没有显式 cleanup 路径**。这是个**已知 tech debt**——失败的 pane 可能成为"僵尸 pane"（不跑 teammate 但占地方）。

用户清理方式：`TeamDelete` 触发 `cleanupTeamDirectories` 会 kill 所有 team 的 pane。

## 9.16 SpawnOutput 返回结构

```ts
export type SpawnOutput = {
  success: boolean
  agentId: string          // 即使失败也返回 (用于 error 关联)
  teamName?: string
  agentName?: string
  paneId?: string          // pane backend 才有
  taskId?: string          // in-process backend 才有
  abortController?: AbortController  // in-process backend 才有
  backendType?: BackendType
  error?: string
  needsIt2Setup?: boolean   // 提示用户装 it2
}
```

字段是 **superset**——3 种 backend 各自填一部分：

| 字段 | in-process | split-pane | separate-window |
|------|:--:|:--:|:--:|
| `success` | ✓ | ✓ | ✓ |
| `agentId` | ✓ | ✓ | ✓ |
| `paneId` | — | ✓ | ✓ |
| `taskId` | ✓ | — | — |
| `abortController` | ✓ | — | — |
| `backendType` | ✓ | ✓ | ✓ |
| `error` | (失败时) | (失败时) | (失败时) |

上层（spawnMultiAgent 调用方）按 `backendType` 字段决定如何处理结果。

## 9.17 spawnMultiAgent 不是真的 "Multi"

注意：**`spawnMultiAgent.ts` 文件名误导**——`spawnTeammate` 函数其实是 **单 teammate spawn**。"Multi" 是因为它被**多次调用**实现 multi-agent 派发，而不是一次调用派多个。

真正的"批量派发"由 caller 实现：

```ts
// AgentTool 等批量派发 caller 推断的伪代码:
const teammates = [
  { name: 'researcher', prompt: '...' },
  { name: 'coder', prompt: '...' },
  { name: 'reviewer', prompt: '...' },
]

const results = await Promise.all(
  teammates.map(t => spawnTeammate(t, context))
)
```

设计原因：
- 单次 spawn 已经够复杂（200+ 行 in-process / 240 行 pane）；
- 并发由 `Promise.all` 处理；
- 每个 teammate 独立失败可控；
- 不需要专门的批量 API。

## 9.18 几个隐性设计判断

### 1. 3 路 spawn 共享前置 helper

3 个 handle* 函数都用 `generateUniqueTeammateName` / `buildInheritedCliFlags` / `assignTeammateColor` / 等共享 helper。重复的部分集中——避免 3 处实现不一致。

### 2. auto mode fallback 是无感降级

`detectAndGetBackend` 失败时 auto mode 自动 fallback 到 in-process——用户不感知。显式 mode 不 fallback——尊重用户选择。

### 3. plan mode 优先于 bypass permission

安全优先——不让 plan mode 被 bypass 绕过。这种 **mode 之间的优先级**是显式规则。

### 4. classifier auto-mode 双重 gate

teammate 继承 `--permission-mode auto` flag，但启动时再走一次 GrowthBook gate——防止"flag 传了但 gate 关了"的不一致。

### 5. 命名冲突自动加后缀

`tester-2` / `tester-3` 自动生成——多 teammate 并发 spawn 时避免冲突。**spawn 函数内部确保唯一**而不是依赖 caller 提前查重。

### 6. bundled mode 强制 self-binary

`isInBundledMode()` 时用 `process.execPath`——确保 teammate 用同 binary，避免版本不一致。

### 7. tmux session detached 创建

`-d` flag 避免突然切走用户视图——尊重用户当前的 terminal 状态。

### 8. waitForPaneShellReady 500ms 经验值

shell 启动延迟——硬编码 500ms 兜底。如果有可靠的 ready signal 会更好（但 send-keys 模型没有）。

### 9. 没有全局 spawn 失败 cleanup

部分失败时 teammate 各自负责——zombie pane 可能残留。`TeamDelete` 时统一清理。这是 known tech debt。

### 10. 返回 SpawnOutput 是 superset

3 种 backend 共用 output 类型——`paneId` / `taskId` 字段二选一存在。caller 按 `backendType` 字段决定使用哪些。**统一类型 + optional 字段**避免泛型类型复杂度。

## 9.19 1093 行复杂度来源

spawnMultiAgent.ts 1093 行包含：

- 3 个独立 spawn handler（in-process / split-pane / separate-window）
- backend detection + fallback 策略
- CLI flag 继承逻辑（5 类 flag）
- tmux session / window 管理
- 命名冲突自动解决
- team file 更新（写 members 数组）
- 颜色分配
- pane 创建 + 装饰（border / title）
- shell 启动等待
- out-of-process task 注册（UI 跟踪）
- 多种错误处理 + logForDebugging
- backendType / paneId / taskId / abortController 各种条件返回

每一项都不算特别复杂，但**协调 3 种 backend × 多个独立子系统**累积到 1093 行。

## 9.20 与其它专题串联

- **[01 激活机制](./01-激活机制与团队配置.md)**：spawn 写 team file 的 members 数组——和 TeamCreate 创建 teamFile 配对；
- **[02 Teammate 身份](./02-Teammate身份与上下文.md)**：spawn 通过 CLI flag 让新进程调 `setDynamicTeamContext`；
- **[03 Pane backend](./03-Pane-backend-tmux-iterm2.md)**：spawn-split-pane 调 PaneBackend 接口创建 pane；
- **[04 In-process backend](./04-In-process-backend.md)**：spawn-in-process 调 spawnInProcessTeammate + startInProcessTeammate；
- **[07 In-process Runner](./07-In-process-Runner.md)**：spawn-in-process 启动的 runner；
- **[agent 02 派发模式](../agent/02-派发模式.md)**：AgentTool 在 swarm 模式下复用 spawnTeammate。

## 9.21 小结

- `spawnMultiAgent.ts` 1093 行 - 单 teammate spawn 入口（文件名误导是历史原因）；
- 3 条 spawn 路径：in-process / split-pane / separate-window；
- auto mode 失败自动 fallback in-process；显式 mode 不 fallback；
- `buildInheritedCliFlags` 5 类 flag 继承：permission mode / model / settings / plugins / chrome；
- plan mode 优先于 bypass——安全优先；
- `generateUniqueTeammateName` 自动加后缀 `-2` / `-3` 解决重名；
- `getTeammateCommand` 3 fallback：env var > process.execPath（bundled）> 'claude'；
- handleSpawnInProcess 6 步、handleSpawnSplitPane 9 步；
- pane 创建后等 500ms 让 shell ready；
- 部分失败常态——单个 spawn 返回 `{success: false, error}` 而不是抛错；
- SpawnOutput 是 superset 类型——3 种 backend 各填一部分字段；
- 真正的"批量派发"由 caller 用 `Promise.all` 实现。

下一篇 → [10 Team Memory 与持久化](./10-Team-Memory与持久化.md)
