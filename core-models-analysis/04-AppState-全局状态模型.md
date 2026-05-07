# 04. AppState — 全局状态模型

> 这一篇做一件事：把 `restored-src/src/state/AppStateStore.ts` 569 行**作为 model**讲透——这个超大型 state model 由 95+ 字段组成，覆盖 settings / 模型 / UI / 任务 / agent / MCP / plugin / hooks / bridge / speculation / classifier 全部子系统。读完之后你能解释"为什么 Claude Code 不把 AppState 拆成多个独立 store"。
>
> 接 02 篇：Tool.ts 的 ToolUseContext 通过 `getAppState()` 拿 AppState；本篇是 AppState 自己的 model。

---

## 1. AppState 在所有 model 里的位置

`AppState` 是 Claude Code 整个进程的**单一全局可变状态**。三个特征：

- **唯一**：每个进程只有一个 root AppState
- **可变**：通过 `setAppState((prev) => next)` 更新
- **不可变更新**：每次更新返回**新对象**（DeepImmutable 强制）——React 友好，diff 友好

它和其它 model 的关系：

- Tool.ts 的 `ToolUseContext.getAppState()` 读它
- Tool.ts 的 `ToolUseContext.setAppState()` 写它
- 每个 React 组件可以通过 hook 订阅其某一切片
- Hooks 系统在 callback 里通过 `HookCallbackContext.getAppState` 读它
- Plugin / MCP 装载结果都进 AppState 的相应子树

打开 `restored-src/src/state/AppStateStore.ts`，569 行。

---

## 2. 文件骨架

```
AppStateStore.ts (1-569)
│
├── 段 0  imports 与小辅助类型              1-79
│        CompletionBoundary / SpeculationResult / SpeculationState
│        FooterItem
│
├── 段 1  AppState 类型定义                89-452     ★ 主体
│        DeepImmutable 部分:               89-158
│            settings / 模型 / UI / 状态线 / kairos /
│            remote / replBridge / footer / 选择
│        Mutable 部分:                     159-452
│            tasks / mcp / plugins / agentDefinitions /
│            fileHistory / attribution / todos /
│            通知 / sessionHooks / tungsten / bagel /
│            chicago MCP / replContext / teamContext /
│            inbox / sandbox / promptSuggestion /
│            speculation / skillImprovement /
│            authVersion / initialMessage /
│            denialTracking / overlays / fastMode /
│            ultraplan
│
├── 段 2  AppStateStore 别名               454
│        type AppStateStore = Store<AppState>
│
└── 段 3  getDefaultAppState               456-569
        生成初始 state
```

我们按"功能分组"而不是"行号顺序"读——95+ 字段按行号读太碎。

---

## 3. 整体结构：DeepImmutable + Mutable 的组合

`AppState` 类型定义起手就分了两段：

```ts
export type AppState = DeepImmutable<{
  // ... 60+ 个字段（行 89-158）
}> & {
  // ... 35+ 个字段（行 159-452）
}
```

**为什么要分？**

注释 89-158 的字段都是**值类型**（boolean、string、number、enum）—— React 友好的浅比较 + 深 immutable。

注释 159-452 的字段大多含**function 类型**（`tasks: { [taskId]: TaskState }`，TaskState 内部有函数；`replContext.console.log: () => void` 等）。`DeepImmutable` 不能正确处理 function（function 不能 readonly）——所以这些字段被排除在 immutable 检查之外。

行 159-160 的注释明确：

```
// Unified task state - excluded from DeepImmutable because TaskState contains function types
tasks: { [taskId: string]: TaskState }
```

**这个设计反映了 TypeScript immutability 的真实约束**——不可能 100% 强制，只能强制能强制的部分，function-bearing object 老老实实标 mutable。

---

## 4. 第一组：核心运行参数（89-95）

```ts
settings: SettingsJson
verbose: boolean
mainLoopModel: ModelSetting
mainLoopModelForSession: ModelSetting
statusLineText: string | undefined
```

最基础的 4 个：

- `settings` — 用户配置全集（`~/.claude/settings.json` + 项目 + flag 合并）
- `verbose` — 是否显示详细输出
- `mainLoopModel` — 当前用的模型（`null` = 默认 / `string` = alias 或全名）
- `mainLoopModelForSession` — 本 session 持久（区别于 turn 级临时切换）
- `statusLineText` — 自定义状态栏文字

#### `mainLoopModel` 和 `mainLoopModelForSession` 的拆分

skill 可以临时切模型（02 篇 5.2 节 `command.model`）—— 这种是 turn 级。如果用户 `/model opus` 显式切 → session 级。

两个字段的存在让"我现在用什么"和"我整个 session 想用什么"能分开追踪——turn 结束后 fall back 到 session 选择，session 结束后 fall back 到全局默认。

---

## 5. 第二组：UI 视图状态（96-108）

```ts
expandedView: 'none' | 'tasks' | 'teammates'
isBriefOnly: boolean
showTeammateMessagePreview?: boolean
selectedIPAgentIndex: number
coordinatorTaskIndex: number
viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
footerSelection: FooterItem | null
```

UI 状态机的位字段。每个枚举对应一种"用户当前在看什么 / 选什么"。

注释 99-103 提到 `coordinatorTaskIndex` 在 AppState 而不是 local state 的原因：

```
AppState (not local) so the panel can read it directly without prop-drilling
through PromptInput → PromptInputFooter.
```

**避免 props 钻孔（prop drilling）**——React 组件树深时，state 提升到 AppState 是简洁的解。

---

## 6. 第三组：权限上下文（109）

```ts
toolPermissionContext: ToolPermissionContext
```

整个权限系统的运行时状态——permission rules、mode、bypass 可用性等。这一字段单独占据一个完整的 model，第 05 篇展开。

放在 AppState 里说明：**权限上下文是会话级的**——某次允许 / 拒绝会更新 toolPermissionContext，影响后续所有 tool 调用。

---

## 7. 第四组：Kairos / Assistant 模式（110-116）

```ts
spinnerTip?: string
agent: string | undefined                  // --agent CLI flag 或 settings
kairosEnabled: boolean                     // assistant 模式总开关
```

注释 113-116 解释 `kairosEnabled`：

```
Assistant mode fully enabled (settings + GrowthBook gate + trust).
Single source of truth - computed once in main.tsx before option
mutation, consumers read this instead of re-calling isAssistantMode().
```

**关键设计**：原本 `isAssistantMode()` 是个函数，每次调用要查 settings + GrowthBook + trust——不便宜。一次性算好放 AppState，**所有消费者直接读字段**。

这是 "**避免重复计算 → 提升到 state**" 的标准模式。

---

## 8. 第五组：Remote Session（117-132）

```ts
remoteSessionUrl: string | undefined
remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
remoteBackgroundTaskCount: number
```

Remote mode（`--remote` flag）的状态。这是 viewer 模式——本地 Claude Code 作为远程 daemon 的 viewer。

`remoteBackgroundTaskCount` 注释 127-131 提到：

```
event-sourced from system/task_started and system/task_notification on the WS.
The local AppState.tasks is always empty in viewer mode — the tasks live in a
different process.
```

**event-sourced**：远程 viewer 里 `AppState.tasks` 是空的（任务跑在另一个进程），但通过 WebSocket 事件计数 task 数。这种"按消息源更新 state"是分布式系统的常见模式。

---

## 9. 第六组：REPL Bridge（133-157）

12 个字段，全部以 `replBridge` 开头：

```ts
replBridgeEnabled / replBridgeExplicit / replBridgeOutboundOnly
replBridgeConnected / replBridgeSessionActive / replBridgeReconnecting
replBridgeConnectUrl / replBridgeSessionUrl
replBridgeEnvironmentId / replBridgeSessionId
replBridgeError / replBridgeInitialName
showRemoteCallout
```

Bridge 是"本地 Claude Code 接受远程 claude.ai 控制"的能力。每个字段表达一个状态机维度——是否 enabled、显式触发还是隐式、是否 outbound only、连接状态、URL 等等。

#### "12 个 boolean / string 描述一个连接状态" 是不是过度设计？

不是。注释清楚地写了每个字段对应的语义：

- `replBridgeEnabled` — 用户**意图**（开关）
- `replBridgeConnected` — 实际**连接成功**
- `replBridgeSessionActive` — 是否有用户在远程使用
- `replBridgeReconnecting` — 是否在重连
- `replBridgeError` — 错误消息（UI 展示）
- `replBridgeExplicit` — 是 `/remote-control` 命令开的还是 config 开的（影响 UI 提示）

**意图 / 实际 / 状态 / 错误是不同维度**——单字段（比如 `bridgeStatus: 'enabled' | 'connected' | 'error'`）建模会让 UI 渲染逻辑变成连串 if-else。每字段表达一个维度让 React 组件能精准订阅。

---

## 10. 第七组：Tasks（159-167）

```ts
tasks: { [taskId: string]: TaskState }
agentNameRegistry: Map<string, AgentId>
foregroundedTaskId?: string
viewingAgentTaskId?: string
companionReaction?: string
companionPetAt?: number
```

Tasks 是 Claude Code 的"后台任务"系统——`Agent` tool 触发的子 agent、远程 task、teammate 都通过这里管理。

`{ [taskId]: TaskState }` 这种 dict 形态而不是数组——按 ID 查找 O(1)，删除 O(1)。

`agentNameRegistry: Map<string, AgentId>` 让 `SendMessage` tool 能按名字（"alice"）路由到 AgentId（uuid）。

---

## 11. 第八组：MCP（173-184）

```ts
mcp: {
  clients: MCPServerConnection[]            // 当前连接的 MCP servers
  tools: Tool[]                             // MCP 提供的 tools
  commands: Command[]                       // MCP 提供的 commands（含 skills）
  resources: Record<string, ServerResource[]>   // MCP 提供的 resources
  pluginReconnectKey: number                // 重连触发键
}
```

#### `pluginReconnectKey: number` 的设计

注释 178-183 解释：

```
Incremented by /reload-plugins to trigger MCP effects to re-run
and pick up newly-enabled plugin MCP servers. Effects read this
as a dependency; the value itself is not consumed.
```

**这是 React 风格的"effect dependency"模式**——某些 React effect 监听 `pluginReconnectKey`，值改了就 re-run。值本身没语义，只是"变了"这个事实有意义。

类似 React 的 `key` prop——通常我们叫它"epoch counter"。

---

## 12. 第九组：Plugins（185-216）

```ts
plugins: {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  commands: Command[]
  errors: PluginError[]                     // ← 见 07 篇
  installationStatus: {
    marketplaces: Array<{ name, status, error? }>
    plugins: Array<{ id, name, status, error? }>
  }
  needsRefresh: boolean                     // ← 在第 13 节展开
}
```

`needsRefresh` 注释 209-215：

```
Set to true when plugin state on disk has changed (background reconcile,
/plugin menu install, external settings edit) and active components are
stale. In interactive mode, user runs /reload-plugins to consume. In
headless mode, refreshPluginState() auto-consumes via refreshActivePlugins().
```

**dirty flag 模式**：磁盘变了就置 true，用户决定何时消费。这种"延迟应用"避免在用户敲命令一半时突然 reload 打断他。

---

## 13. 第十组：Agent definitions / 文件历史 / Attribution（217-220）

```ts
agentDefinitions: AgentDefinitionsResult
fileHistory: FileHistoryState
attribution: AttributionState
todos: { [agentId: string]: TodoList }
```

`todos` 按 `agentId` 分组——每个 agent 维护自己的 TODO 列表。这和 invokedSkills 的 (name, agentId) 复合键设计一致——**agent 边界一直被尊重**。

---

## 14. 第十一组：Notifications & Elicitation（222-228）

```ts
notifications: {
  current: Notification | null              // 正在展示的
  queue: Notification[]                     // 排队的
}
elicitation: {
  queue: ElicitationRequestEvent[]
}
```

`current + queue` 队列模式——一次只显示一个 notification，新来的排队。

`elicitation` 是 MCP -32042 错误的处理——MCP server 通过 elicitation 请求用户提供 URL。

---

## 15. 第十二组：thinking / promptSuggestion / sessionHooks（229-231）

```ts
thinkingEnabled: boolean | undefined        // undefined = 默认（按设置算）
promptSuggestionEnabled: boolean
sessionHooks: SessionHooksState
```

`thinkingEnabled: undefined` 的语义和 `false` 不同——`undefined` 表示"用 default（依赖 settings 和 model）"，`false` 表示"用户显式关闭"。

这种**三态 boolean** 在配置场景常见——避免"我手动设过 false 还是默认就 false"的混淆。

---

## 16. 第十三组：Tungsten / Bagel / Chicago（232-299）

这一段很多字段：

```ts
tungstenActiveSession?: { sessionName, socketName, target }
tungstenLastCapturedTime?: number
tungstenLastCommand?: { command, timestamp }
tungstenPanelVisible?: boolean
tungstenPanelAutoHidden?: boolean

bagelActive?: boolean
bagelUrl?: string
bagelPanelVisible?: boolean

computerUseMcpState?: { ... }   // chicago MCP 相关
```

#### 这一段揭示了 AppState 的"实验性 / 可选字段"哲学

Tungsten（tmux 控制）、Bagel（WebBrowser）、Chicago（computer use MCP）都是**实验性 feature**——可能在某些用户用，某些不用。它们的状态字段全都是 `?:` optional——不启用就 `undefined`，零成本。

注释 257-258 提到 `computerUseMcpState`：

```
Types inlined (not imported from @ant/computer-use-mcp/types) so external
typecheck passes without the ant-scoped dep resolved. Shapes match
`AppGrant`/`CuGrantFlags` structurally — wrapper.tsx assigns via structural
compatibility.
```

**外部用户的 typecheck 不能依赖 ant-only 包**——所以类型 inline。这是"代码外部化前的保护"。

---

## 17. 第十四组：REPL VM context（301-322）

```ts
replContext?: {
  vmContext: import('vm').Context           // Node.js vm 模块
  registeredTools: Map<string, { name, description, schema, handler }>
  console: { log, error, warn, info, debug, getStdout, getStderr, clear }
}
```

`/repl` 命令的状态——一个长期存在的 vm context，让用户在 REPL 里 evaluate 代码。`registeredTools` 让用户在 REPL 里定义 tool，`console` 重定向 stdout/stderr 让 REPL 输出能被捕捉。

---

## 18. 第十五组：Team & Standalone Agent（323-350）

```ts
teamContext?: {
  teamName / teamFilePath / leadAgentId
  selfAgentId? / selfAgentName? / isLeader? / selfAgentColor?
  teammates: { [id]: { name, agentType?, color?, tmuxSessionName, ... } }
}
standaloneAgentContext?: { name, color? }
```

Swarm 模式：多个 Claude Code 进程协作，每个进程是一个 teammate。这一字段记录"我是谁、我的队友是谁、谁是 leader"。

注释 326-329 强调：

```
Self-identity for swarm members (separate processes in tmux panes)
Note: This is different from toolUseContext.agentId which is for in-process subagents
```

**进程级 ID** vs **进程内 subagent ID** 是两个完全不同的概念，分别建模。

---

## 19. 第十六组：Inbox / Sandbox（351-384）

```ts
inbox: { messages: Array<{ id, from, text, timestamp, status, color?, summary? }> }
workerSandboxPermissions: { queue, selectedIndex }
pendingWorkerRequest: { toolName, toolUseId, description } | null
pendingSandboxRequest: { requestId, host } | null
```

Inbox 是 swarm 内成员间消息系统——队友之间的留言。Sandbox 是 worker 进程的网络访问权限请求——leader 决定批准或拒绝。

`pendingWorkerRequest` 类型为 `T | null` 而不是 `T | undefined`——一个细微选择。null 在 JSON 里能正确序列化，undefined 不能。可能是为了 conversation recovery 时的 JSON 持久化。

---

## 20. 第十七组：Prompt Suggestion / Speculation（385-393）

```ts
promptSuggestion: {
  text: string | null
  promptId: 'user_intent' | 'stated_intent' | null
  shownAt: number
  acceptedAt: number
  generationRequestId: string | null
}
speculation: SpeculationState
speculationSessionTimeSavedMs: number
```

`speculation` 是 Claude Code 的预先执行优化——猜测用户下一步输入，提前跑——用 `SpeculationState`（行 58-77）建模成状态机：

```ts
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }     // ★ Mutable ref
      writtenPathsRef: { current: Set<string> }
      boundary: CompletionBoundary | null
      // ... 等
      pipelinedSuggestion?: ...
    }
```

注意 `{ current: ... }` 这种 ref 模式——避免每条 message 触发整个 messages 数组的 spread copy。性能优化。

---

## 21. 第十八组：剩余杂项（394-451）

最后一段：

```ts
skillImprovement: { suggestion: { skillName, updates: [...] } | null }
authVersion: number                         // login/logout 触发刷新
initialMessage: { message, clearContext?, mode?, allowedPrompts? } | null
pendingPlanVerification?: { plan, verificationStarted, verificationCompleted }
denialTracking?: DenialTrackingState
activeOverlays: ReadonlySet<string>
fastMode?: boolean
advisorModel?: string
effortValue?: EffortValue                   // skill 临时改本 turn effort

ultraplanLaunching? / ultraplanSessionUrl? / ultraplanPendingChoice? / ultraplanLaunchPending?
isUltraplanMode?

replBridgePermissionCallbacks?
channelPermissionCallbacks?
```

每一行都是某个具体子系统的状态。整体规模大但每个都局部清晰。

#### `authVersion: number` 的设计

注释 400：

```
Auth version - incremented on login/logout to trigger re-fetching of auth-dependent data
```

又一个 epoch counter——和 `mcp.pluginReconnectKey` 同模式。值本身没语义，**变化**有语义。

#### `activeOverlays: ReadonlySet<string>`

行 420-421 注释：

```
Active overlays (Select dialogs, etc.) for Escape key coordination
```

**用 Set 协调 ESC 键**——多个对话框同时存在时（嵌套对话），ESC 应该关哪个？把所有 active overlay 放进 Set，最近 push 的关掉。

---

## 22. `getDefaultAppState`（456-569）—— 初始 state 构造

```ts
export function getDefaultAppState(): AppState {
  // 计算初始 permission mode（teammate 在 plan 模式下需要 plan）
  const teammateUtils = require('../utils/teammate.js') as ...
  const initialMode: PermissionMode =
    teammateUtils.isTeammate() && teammateUtils.isPlanModeRequired()
      ? 'plan'
      : 'default'

  return {
    settings: getInitialSettings(),
    tasks: {},
    agentNameRegistry: new Map(),
    // ... 60+ 个字段全部初始化
  }
}
```

#### 几个细节

**(a) 用 `require()` 而不是顶层 `import`**

注释 459-462：

```
Use lazy require to avoid circular dependency with teammate.ts
```

`teammate.ts` 可能反过来 import AppStateStore——形成循环。lazy require 把依赖推迟到运行时。这种 trick 在大型 TS 项目常见。

**(b) 所有字段都被显式初始化**

不利用 TS optional 的"未设置等于 undefined"——这样能避免"我以为 undefined 但其实是某种垃圾数据"的隐患。

**(c) 计算字段（initialMode）放在 getDefaultAppState 内部**

不是写常量。因为 mode 取决于运行时（是不是 teammate / 是不是 plan_mode_required）。

---

## 23. `AppStateStore = Store<AppState>`（454）

```ts
export type AppStateStore = Store<AppState>
```

`Store<T>` 来自 `state/store.ts`（34 行的小文件）—— 一个泛型 Store 抽象。AppState 通过它实现 React 订阅。

具体说，Store 提供：

- `get()` — 读当前 state
- `set(updater)` — 更新（不可变）
- `subscribe(listener)` — 订阅变化

React 组件通过 hook 包装这个 Store，按需订阅子树。

---

## 24. 整体设计反思

### 24.1 为什么不拆成多个 store（多个 Provider / 多个 context）？

React 社区长期争论 "global store vs domain stores"。Claude Code 选择了**单一巨型 AppState**，理由可能是：

1. **跨子系统的依赖太多**：tool 执行时既要读 settings、又要读 mcp.tools、又要写 tasks——分多个 store 要管 N 个订阅。
2. **conversation recovery 简单**：单 store 就是一个 JSON，整个序列化 / 反序列化容易。
3. **不可变更新让性能 OK**：DeepImmutable + 子树 selector 让 React 只 re-render 真变化的部分。
4. **类型层一处定义**：所有 state 形状在一个文件里——别人读源码先看 AppState 就懂全局。

代价：

- 文件长（569 行）
- 字段名容易冲突（实际上没有，因为命名前缀化）
- 团队开发要在同一个文件改

但收益更大——尤其对一个**跨子系统耦合很多**的 agent CLI 来说。

### 24.2 命名约定

观察一下：所有字段名都体现归属：

- `replBridge*` 12 个字段
- `tungsten*` 5 个字段
- `bagel*` 3 个字段
- `ultraplan*` 5 个字段

**前缀命名空间** —— TypeScript 没有真正 namespace，所以用前缀。这让 grep / autocomplete 能定位某子系统的全部字段。

### 24.3 字段顺序

**没有严格顺序**——core 字段（settings、verbose）在前，但后面就是按"哪个 feature 加的就追加在末尾"。这是真实演化的痕迹——AppState 是最久存在的文件之一，每个 feature 都加点字段。

---

## 25. AppState 在 model 网里的关系

```
                    ┌──────────────────────────────┐
                    │   AppState (全局 state)      │
                    │   95+ 字段                    │
                    └──┬─────┬─────┬─────┬─────┬──┘
                       │     │     │     │     │
              settings │     │     │     │     │ tasks
              & 模型   │     │     │     │     │ todos
              ↓        │     │     │     │     │ ↓
           SettingsJson│     │     │     │     │ TaskState[id]
           ModelSetting│     │     │     │     │
                       │     │     │     │     │
                       │ permission│     │     │
                       │ context   │     │     │
                       │ ↓         │     │     │
                       │ Permission│     │     │
                       │ model →   │     │     │
                       │ 05 篇      │     │     │
                       │           │     │     │
                       │     mcp:  │     │     │
                       │     ↓     │     │     │
                       │     {tools, commands, │
                       │      resources, ...}  │
                       │           │     │     │
                       │     plugins:    │     │
                       │     ↓           │     │
                       │     LoadedPlugin[] →  │
                       │     07 篇        │     │
                       │                 │     │
                       │           sessionHooks│
                       │           agentDefs   │
                       │           ...         │
                       │                       │
                       └───────────────────────┘
                                    │
                                    │ 通过 ToolUseContext
                                    │ getAppState() / setAppState()
                                    │ 暴露给 tool
                                    ▼
                            ┌──────────────────┐
                            │   每个 Tool 调用  │
                            │   读 / 写 AppState│
                            └──────────────────┘
```

---

## 26. 这一篇你应该带走的几样东西

1. AppState 是 Claude Code 的**单一全局可变 state**——不可变更新（DeepImmutable + 部分 mutable 字段绕过类型限制）
2. 95+ 字段为什么不拆——跨子系统依赖过多，单 store 简单
3. `mainLoopModel` vs `mainLoopModelForSession` 的 turn vs session 双层
4. `kairosEnabled` 是"提升计算到 state"的范例
5. `pluginReconnectKey: number` 和 `authVersion: number` 是 epoch counter——值无意义、变化触发 effect
6. `replBridge*` 12 字段是"意图 / 实际 / 状态 / 错误"多维度建模的范例
7. `notifications: { current, queue }` 是单实例 + 队列模式
8. agent 边界一直被尊重：`todos[agentId]`、`agentNameRegistry`
9. 命名前缀化是替代 namespace 的工具
10. `getDefaultAppState` 用 `require` 打破循环依赖

---

## 27. 设计哲学要点（08 篇会展开）

1. **单一 source of truth**：所有跨子系统的状态在一处。
2. **immutability 是 best-effort**：能 immutable 的强制 immutable，function-bearing 子树排除。
3. **derived state 提升到 state**：`kairosEnabled`、计算结果一次算好缓存为字段。
4. **epoch counter 触发 effect**：值无意义、变化有意义。
5. **三态 optional**：`true / false / undefined` 各表达一种意图。
6. **lifecycle 分层**：turn 级 / session 级 / 持久级各自字段。
7. **agent 维度透传**：所有按 agent 分组的状态用 agentId 复合键。
8. **延迟应用**：dirty flag (`needsRefresh`) 让用户决定何时消费变化。
9. **命名空间通过前缀**：避免 95 个字段命名冲突。

下一篇 05 看 `Permission` —— AppState.toolPermissionContext 那一切片的展开。
