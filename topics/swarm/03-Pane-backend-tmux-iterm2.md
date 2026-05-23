# 03 Pane backend (tmux + iTerm2)

> 多个 teammate 怎么"被用户看见"——pane-based backend 的两个实现：TmuxBackend 764 行、ITermBackend 370 行。本篇拆 PaneBackend 接口、detection 优先级、各自实现的 trick。

## 3.1 为什么需要 backend 抽象

Lead 和 teammate 是独立的 Claude Code 进程（pane backend 路径）——每个进程跑在自己的 terminal 窗口/pane 里。这意味着：

- 用户**视觉**上要能同时看到所有 teammate 的输出 → 需要 split pane；
- Lead 需要**程序化**地往 teammate 的 stdin 发命令 → 需要"向 pane 发 keys"；
- Lead 需要在 teammate shutdown 时**关掉** pane → 需要 kill pane；
- Teammate 多了可能要**隐藏**部分 pane（用户只想看活跃的）→ 需要 hide/show。

不同终端环境提供这些能力的方式不同：

- **tmux**：通用 terminal multiplexer，命令行接口（`tmux split-window -h`、`tmux send-keys`）；
- **iTerm2**：macOS 原生 split，但需要 `it2` CLI 通过 Python API 操控；
- **普通 terminal**（无 tmux 无 iTerm2）：完全没有 pane 概念。

Claude Code 用 `PaneBackend` 接口把这些差异抽象掉——上层代码（spawnMultiAgent 等）不关心底下是哪个。

## 3.2 PaneBackend 接口

`backends/types.ts:42-168` 的 `PaneBackend` 类型有 11 个方法：

```ts
export type PaneBackend = {
  readonly type: BackendType
  readonly displayName: string
  readonly supportsHideShow: boolean

  isAvailable(): Promise<boolean>
  isRunningInside(): Promise<boolean>

  createTeammatePaneInSwarmView(
    name: string,
    color: AgentColorName,
  ): Promise<CreatePaneResult>

  sendCommandToPane(
    paneId: PaneId,
    command: string,
    useExternalSession?: boolean,
  ): Promise<void>

  setPaneBorderColor(paneId, color, useExternalSession?): Promise<void>
  setPaneTitle(paneId, name, color, useExternalSession?): Promise<void>
  enablePaneBorderStatus(windowTarget?, useExternalSession?): Promise<void>

  rebalancePanes(windowTarget: string, hasLeader: boolean): Promise<void>

  killPane(paneId, useExternalSession?): Promise<boolean>
  hidePane(paneId, useExternalSession?): Promise<boolean>
  showPane(paneId, targetWindowOrPane, useExternalSession?): Promise<boolean>
}
```

11 个方法分 3 类：

| 类 | 方法 | 作用 |
|----|------|------|
| **探测** | `isAvailable` / `isRunningInside` | 当前环境能不能用这个 backend |
| **创建与操控** | `createTeammatePaneInSwarmView` / `sendCommandToPane` / `setPaneBorderColor` / `setPaneTitle` / `enablePaneBorderStatus` / `rebalancePanes` | 起 pane、发命令、装饰 UI |
| **生命周期** | `killPane` / `hidePane` / `showPane` | 关闭与隐藏 |

注意 `supportsHideShow: boolean` 是只读字段——告诉上层"我支不支持隐藏"。**tmux 支持，iTerm2 不支持**。上层据此决定能不能用 hide UI。

### useExternalSession 参数

很多方法都有 `useExternalSession?: boolean` 参数。这是 **tmux 特有的概念**：

- `useExternalSession = false`：操作 tmux 在**用户已存在的 tmux session**（用户已经在 tmux 里跑 Claude）；
- `useExternalSession = true`：操作 Claude Code 自己起的**独立 tmux session**（`claude-swarm-$PID` socket，用户没在 tmux 里）。

为什么需要两种？因为如果用户没在 tmux 里跑 Claude，但又想用 swarm——Claude Code 会自己起一个独立 tmux session（绕过用户的环境）。但所有 pane 操作要走这个独立 socket，不能动用户的 tmux。

iTerm2 backend 不用这个参数（永远是用户的 iTerm2 进程）。但接口统一保留，让调用方不需要分支。

### CreatePaneResult

```ts
export type CreatePaneResult = {
  paneId: PaneId
  isFirstTeammate: boolean
}
```

返回 pane id + 是不是第一个 teammate。`isFirstTeammate` 影响 layout 策略：

- 第一个 teammate → 创建 swarm view window，layout 是 leader (30%) | teammate (70%)
- 后续 teammate → 在 teammate 那一侧再 split

不同的 layout 由 `rebalancePanes(windowTarget, hasLeader)` 处理。

## 3.3 backend detection 优先级

`backends/registry.ts:136-254` 的 `detectAndGetBackend` 决定用哪个 backend：

```ts
export async function detectAndGetBackend(): Promise<BackendDetectionResult> {
  await ensureBackendsRegistered()
  if (cachedDetectionResult) return cachedDetectionResult

  const insideTmux = await isInsideTmux()
  const inITerm2 = isInITerm2()

  // Priority 1: 在 tmux 里 → 永远用 tmux (即使在 iTerm2 里跑的 tmux)
  if (insideTmux) {
    return { backend: createTmuxBackend(), isNative: true, needsIt2Setup: false }
  }

  // Priority 2: 在 iTerm2 里 (没在 tmux 里) → 尝试 iTerm2
  if (inITerm2) {
    const preferTmux = getPreferTmuxOverIterm2()  // 用户偏好
    if (!preferTmux) {
      const it2Available = await isIt2CliAvailable()
      if (it2Available) {
        return { backend: createITermBackend(), isNative: true, needsIt2Setup: false }
      }
    }

    // iTerm2 但 it2 不可用 → tmux fallback
    if (await isTmuxAvailable()) {
      return {
        backend: createTmuxBackend(),
        isNative: false,  // 注意 non-native
        needsIt2Setup: !preferTmux,
      }
    }

    throw new Error('iTerm2 detected but it2 CLI not installed...')
  }

  // Priority 3: 都不在 → tmux external session
  if (await isTmuxAvailable()) {
    return { backend: createTmuxBackend(), isNative: false, needsIt2Setup: false }
  }

  // 啥都没有
  throw new Error(getTmuxInstallInstructions())
}
```

详细优先级表：

| 用户环境 | preferTmux? | it2? | tmux? | 选择 | isNative |
|---------|:--:|:--:|:--:|------|:--:|
| 在 tmux 里 | — | — | — | tmux | true |
| 在 iTerm2 里（非 tmux） | false | ✓ | — | iTerm2 | true |
| 在 iTerm2 里 | false | ✗ | ✓ | tmux fallback | **false** (需要 it2 setup) |
| 在 iTerm2 里 | true | — | ✓ | tmux | false |
| 在 iTerm2 里 | — | ✗ | ✗ | **报错** | — |
| 普通终端 | — | — | ✓ | tmux external | false |
| 普通终端 | — | — | ✗ | **报错**（提示装 tmux） | — |

注意：**没有"都不在 + 没 tmux"路径会落到 in-process**——这是上层逻辑（不在 detectAndGetBackend 里）。当 `detectAndGetBackend` 抛错且 `isInProcessEnabled()` 返回 true 时，上层会捕获错误降级到 in-process backend。

### 优先级 1：tmux 优于 iTerm2

注释明确："If inside tmux, always use tmux (even in iTerm2)"。

为什么？因为用户在 iTerm2 里手动启了 tmux session——他们**明显偏好 tmux 操作模型**（否则不会在 iTerm2 里再起 tmux）。Claude Code 尊重这个选择。

### "isNative" 的意义

`isNative: true` 表示 backend 和用户当前 terminal 环境**完全匹配**——pane 出现在用户面前的窗口里。

`isNative: false` 表示降级——pane 在另一个独立的 tmux session 里，用户需要手动 `tmux attach` 才能看到。

`needsIt2Setup: true` 提示用户"安装 it2 CLI 能得到 iTerm2 原生体验"——UI 会显示个 hint。

## 3.4 detection.ts —— 探测细节

### isInsideTmuxSync —— 必须看启动时的 TMUX env

`backends/detection.ts:9-19` 的关键设计：

```ts
// Captured at module load time to detect if the user started Claude from within tmux.
// Shell.ts may override TMUX env var later, so we capture the original value.
const ORIGINAL_USER_TMUX = process.env.TMUX
const ORIGINAL_TMUX_PANE = process.env.TMUX_PANE
```

**模块加载时**就保存 `TMUX` 和 `TMUX_PANE` 环境变量——因为 `Shell.ts`（[bashtool 08](../bashtool/08-执行层与环境.md) 提到）会在 spawn 子进程时覆盖 `TMUX` 让 Bash 命令在 Claude 的 socket 里跑。

如果 `isInsideTmux()` 读 `process.env.TMUX` 的当前值，会被覆盖后的值误导。**只读启动时的快照**是正确做法。

注释里还有一段非常重要的：

```
IMPORTANT: We ONLY check the TMUX env var. We do NOT run `tmux display-message`
as a fallback because that command will succeed if ANY tmux server is running
on the system, not just if THIS process is inside tmux.
```

**故意不**用 `tmux display-message` 做 fallback——因为系统里可能跑着别的 tmux server，但当前进程并不在 tmux 内。`TMUX` 环境变量是**唯一可靠**的判定信号。

### isInITerm2 —— 多信号检测

```ts
export function isInITerm2(): boolean {
  if (isInITerm2Cached !== null) return isInITerm2Cached

  const termProgram = process.env.TERM_PROGRAM
  const hasItermSessionId = !!process.env.ITERM_SESSION_ID
  const terminalIsITerm = env.terminal === 'iTerm.app'

  isInITerm2Cached = termProgram === 'iTerm.app' || hasItermSessionId || terminalIsITerm
  return isInITerm2Cached
}
```

3 个信号 OR 一起：

- `TERM_PROGRAM === 'iTerm.app'`：iTerm2 设的环境变量；
- `ITERM_SESSION_ID` 存在：iTerm2 session 标识；
- `env.terminal === 'iTerm.app'`：utils/env 的判定（可能用 OSAScript 或其它）。

任一为真即认定 iTerm2。**多信号冗余**避免环境变量被意外覆盖。

### isIt2CliAvailable —— 用 'session list' 而不是 '--version'

```ts
export async function isIt2CliAvailable(): Promise<boolean> {
  const result = await execFileNoThrow(IT2_COMMAND, ['session', 'list'])
  return result.code === 0
}
```

注释解释：

> Uses 'session list' (not '--version') because --version succeeds even when the Python API is disabled in iTerm2 preferences — which would cause 'session split' to fail later with no fallback.

**`it2 --version` 总能成功**（只要二进制存在），但 `it2 session list` 需要 **iTerm2 Python API 开启**才能工作。Python API 是 iTerm2 偏好设置里的开关，**默认关闭**。

用 `session list` 探测能提前发现"二进制装了但 API 没开"的失败模式——避免后续 `session split` 时才报错。

这是个**精准探测**——不只是"工具存在"，而是"工具能用"。

## 3.5 TmuxBackend 关键设计

### 双 socket 设计

`TmuxBackend.ts:77-91`：

```ts
function runTmuxInUserSession(args): Promise<...> {
  return execFileNoThrow(TMUX_COMMAND, args)  // 用户的默认 tmux socket
}

function runTmuxInSwarm(args): Promise<...> {
  return execFileNoThrow(TMUX_COMMAND, ['-L', getSwarmSocketName(), ...args])
  // 独立 socket: claude-swarm-{PID}
}
```

`-L <socket-name>` 让 tmux 用一个**独立的 socket 文件**——和用户的 tmux server 隔离。

注释（`TmuxBackend.ts:96-103`）：

```
When running INSIDE tmux (leader is in tmux):
- Splits the current window to add teammates alongside the leader
- Leader stays on left (30%), teammates on right (70%)

When running OUTSIDE tmux (leader is in regular terminal):
- Creates a claude-swarm session with a swarm-view window
- All teammates are equally distributed (no leader pane)
```

两种模式：

| 用户在 tmux 里 | 用户不在 tmux 里 |
|---|---|
| 用 `runTmuxInUserSession` | 用 `runTmuxInSwarm`（独立 socket） |
| 在用户当前 window 里 split | 创建独立 session `claude-swarm` |
| leader (30%) + teammates (70%) | 全部 teammate 均分（无 leader pane） |

为什么用独立 socket 而不是用户的默认 socket？

- **隔离**：用户的 tmux session 不受 swarm 干扰；
- **PID-based**：socket 名含 PID（`claude-swarm-${process.pid}`），多个 Claude 实例不冲突；
- **生命周期**：Claude Code 退出后 socket 自动清理。

### tmux 颜色映射

```ts
function getTmuxColorName(color: AgentColorName): string {
  const tmuxColors: Record<AgentColorName, string> = {
    red: 'red',
    blue: 'blue',
    green: 'green',
    yellow: 'yellow',
    purple: 'magenta',
    orange: 'colour208',  // ANSI 256-color
    pink: 'colour205',
    cyan: 'cyan',
  }
  return tmuxColors[color]
}
```

8 种 agentColor → tmux 颜色字符串映射。注意 `orange` / `pink` 没有标准 tmux 名，用 ANSI 256-color 代码。这种**显式映射表**避免"颜色名拼写不一致"的 bug——agentColor 是 Claude Code 内部命名，tmux 用自己的命名，两者用 mapping 桥接。

### pane creation lock

```ts
async createTeammatePaneInSwarmView(name, color) {
  const releaseLock = await acquirePaneCreationLock()
  try {
    const insideTmux = await this.isRunningInside()
    if (insideTmux) {
      return await this.createTeammatePaneWithLeader(name, color)
    }
    return await this.createTeammatePaneExternal(name, color)
  } finally {
    releaseLock()
  }
}
```

`acquirePaneCreationLock` 保证**多个 teammate 串行创建**——避免并发 split-window 命令导致 layout 混乱。

[bashtool 04](../bashtool/04-permission决策核心.md) 提过的 promise-based mutex 模式在这里又出现一次：

```ts
function acquirePaneCreationLock(): Promise<() => void> {
  let release: () => void
  const newLock = new Promise<void>(resolve => { release = resolve })
  const previousLock = paneCreationLock
  paneCreationLock = newLock
  return previousLock.then(() => release!)
}
```

经典的 "promise-based mutex"——每个调用 await 上一个 lock，并把自己变成下一个 lock。释放是 resolve 当前 promise 让等待者继续。

### sendCommandToPane

```ts
async sendCommandToPane(paneId, command, useExternalSession = false) {
  const runTmux = useExternalSession ? runTmuxInSwarm : runTmuxInUserSession
  const result = await runTmux(['send-keys', '-t', paneId, command, 'Enter'])
  if (result.code !== 0) {
    throw new Error(`Failed to send command to pane ${paneId}: ${result.stderr}`)
  }
}
```

`tmux send-keys -t <pane> <text> Enter` 是 tmux 把字符发给 pane 的标准方式。注意最后的 `Enter` 是模拟回车——必须传，否则只是把字符堆在 pane 但不执行。

发命令到 teammate pane = 模拟用户在键盘上敲字。**没有专门的 IPC 通道**——纯通过模拟终端输入。这是个**极其简单**的协议，但够用。

## 3.6 ITermBackend 关键设计

### 通过 `it2` CLI 操控

```ts
function runIt2(args): Promise<...> {
  return execFileNoThrow(IT2_COMMAND, args)
}
```

`it2` 是 iTerm2 开源的 Python CLI，通过 iTerm2 的 Python API 操控。常见命令：

- `it2 session split` —— 分割 pane；
- `it2 session list` —— 列所有 session；
- `it2 session send-text <session-id> <text>` —— 发文本；
- `it2 session close <session-id>` —— 关 session。

### supportsHideShow = false

```ts
readonly supportsHideShow = false
```

iTerm2 不支持 hide/show pane。原因是 iTerm2 的 split 是窗口级的——隐藏一个 pane 后没法可靠恢复。

这意味着用户用 iTerm2 backend 时，UI 上没有 hide 按钮。**接口的能力声明驱动 UI**——干净的设计。

### 解析 split 输出

```ts
function parseSplitOutput(output: string): string {
  const match = output.match(/Created new pane:\s*(.+)/)
  if (match && match[1]) {
    return match[1].trim()
  }
  return ''
}
```

`it2 session split` 输出格式：`Created new pane: <session-id>`。用 regex 提取 session ID。

注释里有个**已知限制**：

```
NOTE: This UUID is only valid when splitting from a specific session
using the -s flag. When splitting from the "active" session, the UUID
may not be accessible if the split happened in a different window.
```

**iTerm2 的 RPC 接口在跨窗口 split 时不返回可靠 UUID**——所以 ITermBackend 必须**显式指定 source session**。

### 获取 leader 的 session ID

```ts
function getLeaderSessionId(): string | null {
  const itermSessionId = process.env.ITERM_SESSION_ID
  if (!itermSessionId) return null
  const colonIndex = itermSessionId.indexOf(':')
  if (colonIndex === -1) return null
  return itermSessionId.slice(colonIndex + 1)
}
```

`ITERM_SESSION_ID` 格式是 `wXtYpZ:UUID`——前面是 window/tab/pane 索引，冒号后是 session UUID。`it2` CLI 需要 UUID，所以截取冒号后的部分。

这种**协议格式提取**是真实 iTerm2 集成的细节——这种边界处理代码 LLM 工具集成里很多。

## 3.7 detection cache —— 不可变假设

`detection.ts:21-25`：

```ts
let isInsideTmuxCached: boolean | null = null
let isInITerm2Cached: boolean | null = null
```

模块级 cache，**整个进程生命周期不重算**——除非显式调 `resetDetectionCache()`（测试用）。

这是基于一个假设：**用户启动 Claude Code 时所处的终端环境，在进程运行期间不会变**。用户不会启动 Claude 后又切换到 iTerm2 / tmux。

实际是合理的——但**理论上**用户可以用 `tmux attach` 把 Claude 进程 attach 到 tmux 里。这时 detection 是错的——但 swarm 已经选好 backend 不再重测。

这种**"启动时确定，后续不变"** 的设计简化代码，代价是不支持运行时环境变化。

## 3.8 几个隐性设计判断

### 1. backend 注册用 registerXxxBackend 而不是直接 import

`registry.ts` 通过 `registerTmuxBackend` / `registerITermBackend` 接收 backend class——而不是直接 import。原因：

```ts
function createTmuxBackend(): PaneBackend {
  if (!TmuxBackendClass) {
    throw new Error('TmuxBackend not registered. Import TmuxBackend.ts before using the registry.')
  }
  return new TmuxBackendClass()
}
```

**避免循环依赖**——backend 实现 import 自 types/registry，但 registry 又要知道 backend 的 class。用 lazy register 打破循环。

### 2. tmux > iTerm2 是用户偏好的硬编码

"如果在 tmux 里就用 tmux" 没有用户偏好开关。这种**硬编码偏好**简化决策——用户已经在 tmux 里就显然偏好 tmux。

但 "iTerm2 vs tmux" 在 iTerm2 里有 `getPreferTmuxOverIterm2()` 偏好——因为这种情况下用户可能两个都用。

### 3. detection 用启动时 TMUX env 而不是当前

`ORIGINAL_USER_TMUX` 在模块加载时捕获——防 Shell.ts 后续覆盖 TMUX 让探测误判。这是**对全局可变状态的防御性编程**。

### 4. it2 探测用 'session list' 不用 '--version'

精准探测"工具能用"而非"工具存在"。Python API 必须开启才行——`session list` 能验证这个。

### 5. tmux send-keys 当作"模拟用户敲键"

不引入 IPC 通道，纯用 send-keys 发命令。**简单胜过聪明**。

### 6. supportsHideShow 是接口字段而不是方法

接口能力声明驱动 UI——iTerm2 没 hide 能力 → UI 自动不显示 hide 按钮。这种**capability 描述**让上层代码不需要 if/else 区分 backend。

### 7. 独立 socket 隔离用户 tmux

`claude-swarm-${PID}` 让 Claude Code 自己起的 tmux 不污染用户。PID 让多 Claude 实例不冲突。

### 8. PaneId 用 string 不用结构化类型

`type PaneId = string`——tmux 给 `%0`/`%1`，iTerm2 给 UUID。**用同一个 string 类型**让上层不需要区分。

代价：PaneId 失去类型安全（不能区分 tmux pane id 和 iterm2 session id）。实际不需要——上层永远是一个 backend，pane id 都来自同一 backend。

## 3.9 小结

- `PaneBackend` 11 个方法分 3 类：探测 / 创建操控 / 生命周期；
- `useExternalSession` 是 tmux 特有概念——区分用户 tmux session vs Claude 独立 socket；
- detection 5 级优先级：tmux > iTerm2 + it2 > tmux fallback > tmux external > 报错；
- TmuxBackend 用模块加载时的 `TMUX` env 快照——防 Shell.ts 覆盖；
- ITermBackend 用 `it2 session list` 探测——必须 iTerm2 Python API 开启才算可用；
- TmuxBackend 双 socket：用户 session vs `claude-swarm-${PID}` 独立 socket；
- send-keys 模拟用户敲键——没有专门 IPC 通道；
- pane creation lock 防多 teammate 并发 split 冲突；
- iTerm2 不支持 hide/show——靠 `supportsHideShow` 字段声明能力让 UI 自适应；
- backend 注册用 lazy register 打破循环依赖；
- detection 一次缓存——基于"启动时环境不变"假设。

下一篇 → [04 In-process backend](./04-In-process-backend.md)
