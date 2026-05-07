# 02. Tool / ToolDef / ToolUseContext — 工具的统一抽象

> 这一篇做一件事：把 `restored-src/src/Tool.ts` 全部 792 行**作为 model**讲透。Tool 是 Claude Code 里"模型可以调用的能力"的抽象，和 Command（用户/SkillTool 入口）形成对偶关系。这一篇还把 ToolUseContext（执行上下文）一并讲完——它是连接所有 model 的"信封"。
>
> 接 01 篇：Command 是入口的统一形状，Tool 是执行能力的统一形状。两者是 Claude Code agent 系统的**两个支点**。

---

## 1. Tool 在所有 model 里的位置

Claude API 里"工具调用"是个一等公民——模型在 response 里发出 `tool_use` block，宿主执行 tool，把结果作为 `tool_result` 发回。Claude Code 把这套机制做成了**插件化的本地体系**。

具体说：

- 60+ 个内建 tool（Bash / Read / Write / Edit / Glob / Grep / WebSearch / Task / Agent / SkillTool / WebFetch / NotebookEdit / TodoWrite / ... ）
- MCP 协议接入的远程 tool
- LSP 接入的本地工具
- 用户/plugin 自定义的 tool

它们都遵循同一个 `Tool` 接口。这个接口设计得很重——792 行——因为它要解决**所有方面**的问题：

- 如何被模型调用（execute）
- 如何决定能不能调用（permissions）
- 如何被模型 schema 描述（zod / JSON Schema）
- 如何在 UI 渲染（5 种 React 渲染钩子）
- 如何参与并发安全决策
- 如何参与 hooks 系统
- 如何被分类器评估
- 如何被搜索（ToolSearch 工具搜索）

这是这一篇要讲的全部。

---

## 2. 文件总骨架

打开 `restored-src/src/Tool.ts`。 整体可以切成 7 段：

```
Tool.ts (1-792)
│
├── 段 0  imports                        1-88
│
├── 段 1  辅助类型                       90-156
│        ValidationResult / SetToolJSXFn /
│        ToolPermissionContext / CompactProgressEvent
│
├── 段 2  ToolUseContext                 158-300   ★ 信封模型
│        所有 tool 执行时拿到的"巨型 context"
│
├── 段 3  Progress / ToolResult          302-336
│        异步进度报告 + 工具返回值结构
│
├── 段 4  辅助函数 + AnyObject           338-360
│        toolMatchesName / findToolByName / AnyObject
│
├── 段 5  Tool 接口                      362-695   ★ 核心
│        45+ 个字段/方法的"重抽象"
│
├── 段 6  Tools 别名                     697-701
│
└── 段 7  ToolDef + buildTool            703-792
        从部分定义构造完整 Tool（默认值填充）
```

我们按段精读。重点是段 2、段 5、段 7。

---

## 3. 段 1：辅助类型

### 3.1 `ValidationResult`（95-101）

```ts
export type ValidationResult =
  | { result: true }
  | {
      result: false
      message: string
      errorCode: number
    }
```

**两态结果**——成功就只是 `{result: true}`，失败带 message 和 errorCode。

`errorCode` 用 number 不用字符串——很 C 语言风格，但目的不是 i18n（也没什么 i18n），而是 **telemetry 里能直接 group by 错误码**。

每个 tool 自己的 `validateInput` 返回这个。SkillTool 用了 1/2/4/5/6 这些数字（参考 06 篇）——内部约定。

### 3.2 `ToolPermissionContext`（123-138）

```ts
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>
```

注意 `DeepImmutable`——这个 context 是**只读的**，类型层面就强制了。修改它要走"复制 + 修改"模式（看 06 篇 5.5 节 `contextModifier` 的装饰器链就是这么做的）。

`alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules` 三类规则按 source 分组（`ToolPermissionRulesBySource = { [source]?: string[] }`）——这让"清掉某个 source 的所有规则"很容易（比如重新装载 plugin 时清掉 plugin source 那一组）。

第 05 篇会专门展开 PermissionContext。

### 3.3 `CompactProgressEvent`（150-156）

```ts
export type CompactProgressEvent =
  | { type: 'hooks_start'; hookType: 'pre_compact' | 'post_compact' | 'session_start' }
  | { type: 'compact_start' }
  | { type: 'compact_end' }
```

compact（自动总结压缩）三阶段事件。tool 执行时如果触发 compact，会通过 `onCompactProgress` 回调上报。

---

## 4. 段 2：ToolUseContext — Claude Code 最大的"信封"

```ts
export type ToolUseContext = {
  options: { ... }                       // 16 个静态选项
  abortController: AbortController       // 取消信号
  readFileState: FileStateCache          // 文件读取去重
  getAppState(): AppState                // 全局状态读
  setAppState(...): void                  // 全局状态写
  setAppStateForTasks?: ...               // ★ 子 agent 专用
  handleElicitation?: ...                 // MCP -32042 处理
  setToolJSX?: SetToolJSXFn              // UI 注入
  addNotification?: ...                   // OS 通知
  appendSystemMessage?: ...               // UI 系统消息
  sendOSNotification?: ...                // OS 提醒
  nestedMemoryAttachmentTriggers?: Set<string>
  loadedNestedMemoryPaths?: Set<string>
  dynamicSkillDirTriggers?: Set<string>   // ★ 07 篇用过
  discoveredSkillNames?: Set<string>      // 实验性 skill 发现
  userModified?: boolean
  setInProgressToolUseIDs: ...
  setHasInterruptibleToolInProgress?: ...
  setResponseLength: ...
  pushApiMetricsEntry?: ...
  setStreamMode?: ...
  onCompactProgress?: ...
  setSDKStatus?: ...
  openMessageSelector?: ...
  updateFileHistoryState: ...
  updateAttributionState: ...
  setConversationId?: ...
  agentId?: AgentId                       // ★ 子 agent 标识
  agentType?: string
  requireCanUseTool?: boolean
  messages: Message[]                     // 当前 conversation
  fileReadingLimits?: ...
  globLimits?: ...
  toolDecisions?: Map<...>
  queryTracking?: QueryChainTracking      // ★ 嵌套深度
  requestPrompt?: ...                     // 用户交互 prompt
  toolUseId?: string
  criticalSystemReminder_EXPERIMENTAL?: string
  preserveToolUseResults?: boolean
  localDenialTracking?: DenialTrackingState
  contentReplacementState?: ContentReplacementState
  renderedSystemPrompt?: SystemPrompt
}
```

**这是整个文件最大的类型——50 多个字段**。Tool 执行时拿到的"信封"。

### 4.1 这个 model 为什么这么大

不是设计失误。它的本质是**"工具执行环境"的所有维度**。tool 可能需要：

- 读全局状态（`getAppState`）
- 改全局状态（`setAppState`）
- 取消（`abortController`）
- 发消息（`appendSystemMessage` / `addNotification`）
- 渲染 UI（`setToolJSX`）
- 知道自己在哪个 agent 里跑（`agentId`）
- 知道自己嵌套多深（`queryTracking.depth`）
- 限速（`fileReadingLimits` / `globLimits`）
- 报告进度（`onCompactProgress` / `setStreamMode`）
- 记录决策（`toolDecisions`）
- 等等

每个都是必要的，没有冗余。但**没有 tool 用得到全部**——这就是为什么大量字段是 `?:` optional。tool 实现按需取用。

### 4.2 几个值得单独看的字段

#### `options.commands: Command[]`（160）

ToolUseContext 持有命令表。这就是 SkillTool 怎么找到 skill 的——`context.options.commands.find(...)`。这条引用让 Tool 和 Command 模型间形成连接。

#### `agentId?: AgentId`（245）

只在子 agent 里被设置。注释明确：

```
Only set for subagents; use getSessionId() for session ID.
Hooks use this to distinguish subagent calls.
```

这是判断"我现在是不是在 fork 子 agent 里跑"的关键。子 agent 的 invokedSkills、telemetry 都按 agentId 分组。

#### `messages: Message[]`（250）

当前 conversation 的全部消息。tool 可以查历史——比如 SkillTool 看是否已经 load 过 skill（避免重复）。

#### `setAppStateForTasks?`（185-192）

```ts
setAppStateForTasks?: (f: (prev: AppState) => AppState) => void
```

注释 184-191：

```
Always-shared setAppState for session-scoped infrastructure (background
tasks, session hooks). Unlike setAppState, which is no-op for async agents
(see createSubagentContext), this always reaches the root store so agents
at any nesting depth can register/clean up infrastructure that outlives
a single turn.
```

**有意思的点**：`setAppState` 在子 agent 里**是 no-op**——子 agent 的 state 修改不该污染主会话。但有些 infra（session hooks、background task）要跨 agent 边界生效。这个**双 setAppState 设计**让两个语义并存。

#### `queryTracking?: QueryChainTracking`（266）

```ts
export type QueryChainTracking = {
  chainId: string
  depth: number
}
```

**嵌套深度追踪**。模型可以发起 SkillTool（嵌套调 skill），skill 可以触发 Agent（嵌套子 agent），子 agent 又可以发 SkillTool......depth 标记嵌套层级。telemetry 按它分类（`'nested-skill' vs 'claude-proactive'`）。

#### `contentReplacementState?: ContentReplacementState`（287-292）

注释 285-291：

```
Per-conversation-thread content replacement state for the tool result
budget. When present, query.ts applies the aggregate tool result budget.
Main thread: REPL provisions once (never resets — stale UUID keys
are inert). Subagents: createSubagentContext clones the parent's state
by default (cache-sharing forks need identical decisions), or
resumeAgentBackground threads one reconstructed from sidechain records.
```

**长会话里 tool result 总和不能无限增长**（context window 有限）。这个 state 追踪"哪些 tool result 被替换成了 `<file path=...>` 占位符"。子 agent 默认克隆父的——这样 fork 时 prompt cache 不会因为状态分歧而被打掉。

#### `renderedSystemPrompt?: SystemPrompt`（295-299）

注释 293-298：

```
Parent's rendered system prompt bytes, frozen at turn start.
Used by fork subagents to share the parent's prompt cache — re-calling
getSystemPrompt() at fork-spawn time can diverge (GrowthBook cold→warm)
and bust the cache.
```

**system prompt 在 fork 时被冻结传递**。原因：子 agent 启动时再算一次 system prompt 可能产生轻微差异（GrowthBook flag 在 fork 时刚好被刷新），结果 prompt cache 命中失败，fork 子 agent 要重新付完整 prefix 缓存费——浪费几千 token。

这个字段的存在反映了**性能优化已经到了"防止 GrowthBook 状态变化导致 cache miss"这种细粒度**。

---

## 5. 段 3：Progress 和 ToolResult

### 5.1 `Progress`（305-310）

```ts
export type Progress = ToolProgressData | HookProgress

export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}
```

tool 执行中可以多次回调上报进度。`data` 是按 tool 类型 narrow 的——比如 `BashProgress` 是 stdout 流，`AgentToolProgress` 是子 agent 的消息。

### 5.2 `ToolResult<T>`（321-336）

```ts
export type ToolResult<T> = {
  data: T
  newMessages?: (UserMessage | AssistantMessage | AttachmentMessage | SystemMessage)[]
  contextModifier?: (context: ToolUseContext) => ToolUseContext
  mcpMeta?: {
    _meta?: Record<string, unknown>
    structuredContent?: Record<string, unknown>
  }
}
```

#### 三个字段三种"副作用类型"

- `data` — tool 自己的返回数据（必需）
- `newMessages` — tool 跑出来要注入到 conversation 的消息（可选）
- `contextModifier` — 修改主循环 context（可选）

`newMessages` 让 tool 能"在 conversation 里留痕"。比如 SkillTool inline 模式把 skill body 作为 user message 注入——它就是放在 newMessages 里。

`contextModifier` 让 tool 能"改后续 turn 的执行环境"。SkillTool 用它把 `allowedTools` 写回 always-allow rules（参考 06 篇 5.5）。**只有"非并发安全"的 tool 才能用**（注释 329 说），因为并发场景下多个 tool 同时改 context 会乱。

`mcpMeta` 是 MCP 协议透传——`structuredContent` 让 SDK consumer 拿到结构化数据，`_meta` 是自定义元数据。

---

## 6. 段 5：Tool 接口（362-695）— 重型核心

`Tool` 接口本身约 330 行——必须分组看。

### 6.1 类型参数（362-366）

```ts
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = { ... }
```

三个参数：

- `Input` — 输入 schema（zod object）
- `Output` — 输出类型
- `P` — 进度数据类型（默认通用）

每个具体 tool 给出自己的 `Input/Output/P`，类型层面就有完全的类型推导。

### 6.2 身份与查找（371, 378, 449, 454-456）

```ts
aliases?: string[]
searchHint?: string                   // 给 ToolSearch 关键词检索用
readonly shouldDefer?: boolean        // 默认是否需要 ToolSearch 才能调
readonly alwaysLoad?: boolean         // 永不 defer，turn 1 就发给模型
mcpInfo?: { serverName: string; toolName: string }
readonly name: string
```

#### 这一组揭示了 ToolSearch 体系

Claude Code 把 tool 分成**"立即可见 / 需搜索 / 永远显式"**：

- `alwaysLoad: true` — 像 Read/Bash 这种基础工具，模型 turn 1 必须知道（注释 443-448 强调"turn 1 没有 ToolSearch round-trip"）
- 默认 — 受 ToolSearch 控制
- `shouldDefer: true` — 主动默认 deferred，需要 ToolSearch 才能调

这是一个**控制 prompt 大小**的机制。不能让 60+ tool 全塞进 turn 1 的 system prompt。

### 6.3 执行入口（379-385）

```ts
call(
  args: z.infer<Input>,
  context: ToolUseContext,
  canUseTool: CanUseToolFn,
  parentMessage: AssistantMessage,
  onProgress?: ToolCallProgress<P>,
): Promise<ToolResult<Output>>
```

5 个参数都重要：

- `args` — 类型是 `z.infer<Input>`，从 schema 推导
- `context` — 巨型 ToolUseContext
- `canUseTool` — 权限检查回调（用于嵌套权限）
- `parentMessage` — 触发本次 tool_use 的 assistant message（拿它的 ID 关联子消息）
- `onProgress` — 异步进度回调

### 6.4 描述（386-393）

```ts
description(
  input: z.infer<Input>,
  options: {
    isNonInteractiveSession: boolean
    toolPermissionContext: ToolPermissionContext
    tools: Tools
  },
): Promise<string>
```

注意是个**函数**，不是字段。原因：tool 描述可能依赖 input（比如 Bash 的描述要含命令名）和 context（非交互模式下描述可能不一样）。

### 6.5 Schema（394-400）

```ts
readonly inputSchema: Input                                  // zod
readonly inputJSONSchema?: ToolInputJSONSchema               // MCP 直接给 JSON Schema
outputSchema?: z.ZodType<unknown>
```

**双 schema 选择**：

- 普通 tool 用 zod（`inputSchema`）
- MCP tool 可以直接用 JSON Schema（`inputJSONSchema`）——MCP 协议本来就是 JSON Schema，转 zod 反而麻烦

`outputSchema` 是 optional 的，因为不是所有 tool 都强制 schema 化输出（注释 398 说 "TungstenTool 不定义"）。

### 6.6 行为属性（401-435）

```ts
inputsEquivalent?(a, b): boolean      // 两个 input 等价吗（去重用）
isConcurrencySafe(input): boolean      // 能并发跑吗
isEnabled(): boolean                    // 是否启用
isReadOnly(input): boolean             // 只读？
isDestructive?(input): boolean          // 不可逆操作？
interruptBehavior?(): 'cancel' | 'block'  // 用户新输入时怎么办
isSearchOrReadCommand?(input): { isSearch, isRead, isList? }   // UI 折叠
isOpenWorld?(input): boolean           // 输出域开放（影响安全）
requiresUserInteraction?(): boolean    // 需要用户交互
isMcp?: boolean
isLsp?: boolean
```

每个布尔（或函数返回布尔）都是 tool 的**一个性质维度**，被不同子系统读取：

- `isConcurrencySafe` — query 调度器决定能不能并行
- `isReadOnly` — 文件历史 / undo 系统
- `isDestructive` — UI 警告 / hooks 拦截
- `interruptBehavior` — 用户敲新输入时停 tool 还是排队
- `isSearchOrReadCommand` — UI 折叠成单行还是展开
- `isOpenWorld` — 安全分类器对待方式

**这种"按维度建模"是非常成熟的设计**——把 tool 的多面性显式建模出来，比起"哪里要哪里 if-else"健康得多。

### 6.7 资源消耗（466）

```ts
maxResultSizeChars: number
```

注释 458-465：

```
Maximum size in characters for tool result before it gets persisted to disk.
When exceeded, the result is saved to a file and Claude receives a preview
with the file path instead of the full content.
```

**超过这个字符数，tool result 会被持久化到磁盘**，模型只看到文件路径。这是控制 conversation token 不爆炸的硬约束。

`Read` tool 设 `Infinity`（因为如果 Read 结果都被持久化，就形成 Read→file→Read 死循环）。

### 6.8 Validation 和 Permission（489-516）

```ts
validateInput?(input, context): Promise<ValidationResult>

checkPermissions(input, context): Promise<PermissionResult>

getPath?(input): string

preparePermissionMatcher?(input): Promise<(pattern: string) => boolean>
```

#### 三层校验设计

1. **`validateInput`** — "input 形式上对吗？"（skill 名是否存在、disableModelInvocation 是否设置等）
2. **`checkPermissions`** — "用户/策略允许跑吗？"（deny/allow rules、自动放行、ask 用户）
3. **`call`** — 执行

只有 1 通过才走 2，2 决定 allow 后才走 3。这是**安全分层**——形式错误优先发现，业务允许判断在后。

`getPath` 给"涉及文件路径的 tool"（FileRead/Write/Edit）用，让权限系统能按路径模式匹配。

`preparePermissionMatcher` 注释 509-513：

```
Prepare a matcher for hook `if` conditions (permission-rule patterns like
"git *" from "Bash(git *)"). Called once per hook-input pair; any
expensive parsing happens here. Returns a closure that is called per
hook pattern.
```

**两阶段匹配**——一次性 parse 输入（贵），返回闭包按 pattern 重复匹配（便宜）。这是性能优化。

### 6.9 Prompt（518-523）

```ts
prompt(options: {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  tools: Tools
  agents: AgentDefinition[]
  allowedAgentTypes?: string[]
}): Promise<string>
```

这是给**模型看的 tool description**——告诉模型这个 tool 怎么用、什么时候用、有什么注意事项。

**注意它是函数**，因为 prompt 可能依赖：

- 当前 permission context（"在 plan 模式下你不能用我"）
- tools 列表（"你也可以用 X / Y 来做这个"）
- agents 列表（"这个 tool 派给 X agent 比较合适"）

模型看到的 system prompt 里每个 tool 都有自己的 prompt 段——SkillTool 的 prompt 就在 `tools/SkillTool/prompt.ts`。

### 6.10 用户视图（524-527）

```ts
userFacingName(input): string
userFacingNameBackgroundColor?(input): keyof Theme | undefined
```

UI 展示用。`userFacingName` 默认是 `name`，但可以按 input 变化（"Bash" → "Bash · git status"）。

### 6.11 渲染钩子（566-694）— 五种

最后这一大段是 React 渲染钩子。tool 实现时**只需要定义需要自定义的几个**，其它走默认。

| 钩子 | 时机 |
|---|---|
| `renderToolUseMessage` | tool_use 触发瞬间（input 可能还在 streaming） |
| `renderToolUseProgressMessage?` | tool 执行中（onProgress 回调） |
| `renderToolUseQueuedMessage?` | tool 在队列等待 |
| `renderToolResultMessage?` | tool 完成，渲染结果 |
| `renderToolUseRejectedMessage?` | 用户拒绝权限 |
| `renderToolUseErrorMessage?` | 报错 |
| `renderGroupedToolUse?` | 多个并行 tool_use 合并展示 |
| `renderToolUseTag?` | 在 tool use 后追加标签（timeout、model 等） |

**这种"按状态分钩子"是 React 的标准做法**——不同状态不同 UI。tool 实现时**有默认 fallback**，所以小 tool 不用全实现。

#### `extractSearchText`（599）

注释 580-598 很长。简单说：transcript 搜索的索引文本必须**等于实际渲染的可见文本**——不然就出现"搜索能搜到但屏幕找不到"的 phantom 现象，或反过来"屏幕有但搜不到"的 under-count。这个钩子让每个 tool 显式声明"我的搜索文本是什么"，避免猜。

这一处揭示了 Claude Code 对 UI 一致性的要求——细到搜索匹配。

---

## 7. 段 7：ToolDef + buildTool — "默认值填充" 模式

```ts
type DefaultableToolKeys =
  | 'isEnabled' | 'isConcurrencySafe' | 'isReadOnly' | 'isDestructive'
  | 'checkPermissions' | 'toAutoClassifierInput' | 'userFacingName'

export type ToolDef<...> = Omit<Tool<...>, DefaultableToolKeys> &
  Partial<Pick<Tool<...>, DefaultableToolKeys>>

export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,
    userFacingName: () => def.name,
    ...def,
  } as BuiltTool<D>
}
```

#### 这个模式解决什么

定义一个 tool 完整需要 45+ 个字段——大多数 tool 用默认就够了。`buildTool` 让你**只写需要的字段**，其它走默认。

defaults（757-769）：

```ts
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,        // 默认非安全
  isReadOnly: () => false,                // 默认会写
  isDestructive: () => false,             // 默认非破坏性
  checkPermissions: (input) =>            // 默认 allow + 透传 input
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: () => '',        // 默认跳过分类器
  userFacingName: () => '',
}
```

#### "fail-closed where it matters" — 默认值的方向选择

注释 748-756 写得很到位：

```
Defaults (fail-closed where it matters):
- isEnabled → true
- isConcurrencySafe → false (assume not safe)
- isReadOnly → false (assume writes)
- isDestructive → false
- checkPermissions → { behavior: 'allow', updatedInput }
- toAutoClassifierInput → '' (skip classifier — security-relevant tools must override)
- userFacingName → name
```

为什么 `isConcurrencySafe` 默认 `false`？因为如果作者忘了实现，**保守假设不安全**——不会并行跑导致 race。

为什么 `isReadOnly` 默认 `false`？同理——保守假设会写，让 hooks 系统拦截。

但 `isDestructive` 默认 `false`——这个不保守。注释解释 "Only set when the tool performs irreversible operations (delete, overwrite, send)"。这是因为 destructive 默认 true 会让所有 tool 都触发额外 UI 警告——噪音过多。

**这种"保守 vs 不保守"的方向选择反映了对真实使用场景的理解**——不是无脑保守。

### 7.1 BuiltTool 类型映射（735-741）

```ts
type BuiltTool<D> = Omit<D, DefaultableToolKeys> & {
  [K in DefaultableToolKeys]-?: K extends keyof D
    ? undefined extends D[K]
      ? ToolDefaults[K]
      : D[K]
    : ToolDefaults[K]
}
```

这段类型体操在做什么：**对每个可默认的 key**，

- 如果 D 提供了（且不是 undefined）→ 用 D 的类型
- 否则 → 用 default 的类型

效果：buildTool 返回的类型**精确反映**作者实际写了哪些字段、用了哪些默认。这让后续消费方（特别是 SkillTool / FileReadTool 等导出）能保留类型精度。

注释 735-740 强调这是 "Type-level spread mirroring `{ ...TOOL_DEFAULTS, ...def }`"——运行时和类型层完全一致。

---

## 8. 整套 Tool 在 model 网里的关系

```
                  ┌──────────────────────┐
                  │   Tool / ToolDef     │
                  │  (Tool.ts 接口)       │
                  └──────────┬───────────┘
                             │
            ┌────────────────┼─────────────────┐
            │                │                 │
            ▼                ▼                 ▼
       内建 tool       MCP tool         LSP tool
       (~60 个)         (远程)           (远程)
            │                │                 │
            └────────┬───────┴─────────────────┘
                     ▼
              ┌─────────────────────┐
              │  query.ts 调度器    │
              │  按 isConcurrencySafe│
              │  决定并行/串行       │
              └─────────────────────┘
                     │
                     │ 每次调用传给 tool 的：
                     ▼
              ┌─────────────────────┐
              │  ToolUseContext     │  ← 信封 model
              │  (50+ 字段)          │
              └─────────────────────┘
                     │
                     │ 含字段：
                     ├──→ getAppState() / setAppState() → AppState model
                     ├──→ messages: Message[] → Message model
                     ├──→ options.commands → Command model
                     ├──→ options.tools → Tool model (递归)
                     ├──→ agentId / queryTracking → 子 agent 维度
                     └──→ 各种 setter → UI / state / 任务管理
                     │
                     ▼
              tool.call(args, ctx, ...) → ToolResult<T>
                                              │
                                              ├ data: T
                                              ├ newMessages → 注入 conversation
                                              ├ contextModifier → 修改后续 ctx
                                              └ mcpMeta → MCP 协议透传

                  ┌──────────────────────┐
                  │ tool.checkPermissions │
                  └──────────┬───────────┘
                             │
                             ▼ 返回
              ┌─────────────────────┐
              │  PermissionResult    │  ← Permission model
              │  allow/deny/ask/     │
              │  passthrough         │
              └─────────────────────┘
```

**Tool 在 model 网里和 4 个 model 强耦合**：Command（通过 ctx.options.commands）、Message（输入历史 + 输出注入）、AppState（读写）、Permission（检查决策）。

---

## 9. 这一篇你应该带走的几样东西

读到这里，你应该能：

1. 解释 Tool 接口为什么有 45+ 字段——按"模型如何调用 / 权限 / schema / UI / 行为属性 / 资源 / 安全"分组就清楚了
2. ToolUseContext 这个 50+ 字段的 model 为什么不算"过度设计"
3. `setAppState` 和 `setAppStateForTasks` 的差别为什么有意义（子 agent 边界）
4. `ToolResult` 三种副作用（data / newMessages / contextModifier）的对应场景
5. 三层校验（`validateInput` / `checkPermissions` / `call`）的次序和职责
6. `prompt` 为什么是函数而不是字段
7. 5 种 React 渲染钩子按状态分的设计
8. `buildTool` + `TOOL_DEFAULTS` 的"fail-closed where it matters"原则
9. `alwaysLoad` / `shouldDefer` 揭示的 ToolSearch 分级体系

---

## 10. 设计哲学要点（08 篇会展开）

1. **接口即合约**：Tool 接口大但每个字段都有明确职责，没有 `data: any` 这种逃避抽象。
2. **多维属性建模**：`isReadOnly` / `isDestructive` / `isConcurrencySafe` / `isOpenWorld` 都是独立维度，不强制相互关联。
3. **类型即文档**：`DeepImmutable`、`z.infer<Input>`、`BuiltTool<D>` 这些类型操作在编译期表达"哪里能改 / 哪里不能"。
4. **fail-closed 默认**：保守假设——`isReadOnly: false`、`isConcurrencySafe: false`，让作者**显式 opt in** 弱化约束。
5. **巨型 context = 信封**：ToolUseContext 不是一个函数的参数，而是"运行环境"——所有 tool 在这个环境里跑。承认现实而不是假装能用 5 个参数搞定。
6. **prompt-as-function**：tool description 依赖运行时——是函数不是常量。
7. **状态分钩子的 UI 约定**：renderToolUse / Progress / Queued / Result / Error / Rejected 都有独立钩子。

下一篇 03 看 `Message`——Tool 的输入输出都依赖它，但它的类型定义在快照里**缺失**，得从使用端反推。
