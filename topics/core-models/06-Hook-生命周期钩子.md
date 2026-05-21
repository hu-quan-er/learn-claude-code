# 06. Hook — 生命周期钩子模型

> 这一篇做一件事：把 `restored-src/src/types/hooks.ts` 290 行 + `entrypoints/sdk/coreTypes.ts:25-53` 的 27 种 HookEvent 合起来讲透——HookCallback、HookCallbackMatcher、HookJSONOutput（sync / async）、HookResult、AggregatedHookResult、PermissionRequestResult、HookProgress 这一整套类型体系。读完之后你能解释"为什么 hook 系统能既挡 tool 调用、又能注入 prompt、又能改权限决策"。
>
> 接 02 + 05 篇：Tool 接口里有 hook-related 字段（`preparePermissionMatcher`），Permission 的 reason 之一是 `hook`。本篇是 hook 自己的 model。

---

## 1. Hook 在所有 model 里的位置

Hook 是 Claude Code 的**生命周期扩展点**——让用户在特定时刻（tool 调用前、模型响应后、配置变化时、文件改变时...）插入自定义 shell 命令或 callback，影响主流程。

具体能力：

- **拦截 tool 调用**（PreToolUse）—— 阻止某个 tool 跑，或修改 input
- **追加上下文**（PostToolUse / UserPromptSubmit / SessionStart）—— 给模型注入额外信息
- **决策权限**（PreToolUse 的 permissionDecision）—— hook 可以替代 default 权限引擎决定 allow/deny
- **响应事件**（FileChanged / WorktreeCreate / ConfigChange）—— 文件 / 工作目录 / 配置变化时跑命令
- **业务集成**（Stop / SessionEnd）—— 会话结束时清理、上报、归档

设计上 hook 是 **"声明式 + 程序化"** 双轨：

- 用户 settings 里写**声明式**配置：`{ hooks: { PreToolUse: [{ matcher: 'Bash(git:*)', hooks: [...] }] } }`
- 内部代码注册**程序化** callback：`registerSkillHooks` / `registerPostSamplingHook`

两者都用同一套 model（HookCallback）表达。

---

## 2. 文件骨架

```
types/hooks.ts (1-290)
│
├── 段 0  imports + isHookEvent              1-24
│
├── 段 1  Prompt elicitation 协议            26-47
│        promptRequestSchema / PromptRequest / PromptResponse
│
├── 段 2  syncHookResponseSchema             50-166     ★ Hook 输出的 Zod schema
│        含 hookSpecificOutput 13 种事件特化
│
├── 段 3  hookJSONOutputSchema               169-200
│        union: async hook + sync hook
│
├── 段 4  HookCallbackContext / HookCallback  202-232
│        callback 形态 + matcher
│
├── 段 5  HookProgress / HookBlockingError   234-246
│
├── 段 6  PermissionRequestResult            248-258
│
└── 段 7  HookResult / AggregatedHookResult   260-290
        hook 执行结果（单 + 聚合）
```

`entrypoints/sdk/coreTypes.ts:25-53` 还定义了 `HOOK_EVENTS` 数组——27 种事件名。

---

## 3. HOOK_EVENTS — 27 种事件

```ts
export const HOOK_EVENTS = [
  'PreToolUse', 'PostToolUse', 'PostToolUseFailure',
  'Notification', 'UserPromptSubmit',
  'SessionStart', 'SessionEnd',
  'Stop', 'StopFailure',
  'SubagentStart', 'SubagentStop',
  'PreCompact', 'PostCompact',
  'PermissionRequest', 'PermissionDenied',
  'Setup',
  'TeammateIdle', 'TaskCreated', 'TaskCompleted',
  'Elicitation', 'ElicitationResult',
  'ConfigChange',
  'WorktreeCreate', 'WorktreeRemove',
  'InstructionsLoaded',
  'CwdChanged', 'FileChanged',
] as const
```

按"触发时机"分组：

### 3.1 Tool 生命周期（5）

| 事件 | 时机 |
|---|---|
| `PreToolUse` | tool 即将执行前 |
| `PostToolUse` | tool 执行成功后 |
| `PostToolUseFailure` | tool 执行失败后 |
| `PermissionRequest` | 系统要弹权限对话框前 |
| `PermissionDenied` | 用户拒绝权限后 |

`PreToolUse` 是最强的——能**阻止 tool 跑**、修改 input、影响权限决策。

### 3.2 Session 生命周期（4）

| 事件 | 时机 |
|---|---|
| `Setup` | Claude Code 启动期一次性 |
| `SessionStart` | 每次新 session 开始（含 resume） |
| `SessionEnd` | session 结束 |
| `InstructionsLoaded` | 系统提示加载完成 |

`Setup` vs `SessionStart`：前者是进程级（一次），后者是 session 级（resume 也算）。

### 3.3 Subagent / Task 生命周期（5）

| 事件 | 时机 |
|---|---|
| `SubagentStart` | 子 agent 开始 |
| `SubagentStop` | 子 agent 结束 |
| `TaskCreated` | 任务创建 |
| `TaskCompleted` | 任务完成 |
| `TeammateIdle` | swarm 队友空闲 |

### 3.4 Compact 生命周期（2）

| 事件 | 时机 |
|---|---|
| `PreCompact` | 压缩开始前 |
| `PostCompact` | 压缩结束后 |

让用户能在压缩前后 hook，比如保存某段对话到外部、统计被压缩内容长度等。

### 3.5 用户输入 / 中断（2）

| 事件 | 时机 |
|---|---|
| `UserPromptSubmit` | 用户敲了输入提交后 |
| `Stop` | 模型停止响应（ end_turn） |
| `StopFailure` | 停止时出错 |

`UserPromptSubmit` 也很常用——用户每说一句话都能 hook 处理，比如自动注入"今天日期是 X"这种上下文。

### 3.6 通知 / 交互（2）

| 事件 | 时机 |
|---|---|
| `Notification` | 系统通知（OS 级） |
| `Elicitation` | MCP elicitation 请求 |
| `ElicitationResult` | elicitation 完成 |

### 3.7 配置 / 文件变化（4）

| 事件 | 时机 |
|---|---|
| `ConfigChange` | settings.json / skill 文件变化（08 篇 hot reload 用过这个） |
| `CwdChanged` | working directory 切换 |
| `FileChanged` | watched 文件变化 |
| `WorktreeCreate` / `WorktreeRemove` | git worktree 操作 |

### 3.8 27 种事件的设计哲学

为什么这么多？因为 **hook 的本意是"暴露所有可观察 / 可介入的时刻"**。每个事件让用户**精确订阅**自己关心的时刻——不需要的事件不影响。

如果只暴露少数粗粒度事件（"BeforeAnything" / "AfterAnything"），用户要写复杂的 if-else 判断"这次是什么场景"——不优雅。每事件一个钩子让用户的 hook 命令本身可以简单。

---

## 4. 段 1：Prompt Elicitation 协议（26-47）

```ts
export const promptRequestSchema = lazySchema(() =>
  z.object({
    prompt: z.string(),                        // request id
    message: z.string(),                       // 给用户看的提问
    options: z.array(
      z.object({
        key: z.string(),                       // 选项 ID
        label: z.string(),                     // 选项显示名
        description: z.string().optional(),
      }),
    ),
  }),
)

export type PromptRequest = z.infer<ReturnType<typeof promptRequestSchema>>

export type PromptResponse = {
  prompt_response: string                      // request id
  selected: string                             // 选了哪个 key
}
```

#### 这是 hook 反过来"问用户问题"的协议

某些 hook 在执行中需要用户决策——比如"是否清理临时文件？" hook 通过这个 schema 发起 prompt，UI 弹选择对话框，用户选完返回 `PromptResponse`。

注释 27-28 说明：

```
The `prompt` key acts as discriminator (mirroring the {async:true} pattern),
with the id as its value.
```

`prompt` 字段同时是"判别符"和"id"——这种"双用字段"在 JSON 协议里能省一个字段。

---

## 5. 段 2：syncHookResponseSchema（50-166）— Hook 输出的 Zod Schema

这是 hook 命令 stdout 必须输出的 JSON 结构（sync 形式）：

```ts
syncHookResponseSchema = z.object({
  continue: z.boolean().optional(),                  // 是否继续主流程
  suppressOutput: z.boolean().optional(),            // 隐藏 stdout
  stopReason: z.string().optional(),                 // continue=false 时的停止原因
  decision: z.enum(['approve', 'block']).optional(), // 决策
  reason: z.string().optional(),                     // 决策解释
  systemMessage: z.string().optional(),              // 给用户的警告
  hookSpecificOutput: z.union([                      // ← 13 种事件特化输出
    z.object({ hookEventName: z.literal('PreToolUse'), ... }),
    z.object({ hookEventName: z.literal('UserPromptSubmit'), ... }),
    z.object({ hookEventName: z.literal('SessionStart'), ... }),
    z.object({ hookEventName: z.literal('Setup'), ... }),
    z.object({ hookEventName: z.literal('SubagentStart'), ... }),
    z.object({ hookEventName: z.literal('PostToolUse'), ... }),
    z.object({ hookEventName: z.literal('PostToolUseFailure'), ... }),
    z.object({ hookEventName: z.literal('PermissionDenied'), ... }),
    z.object({ hookEventName: z.literal('Notification'), ... }),
    z.object({ hookEventName: z.literal('PermissionRequest'), ... }),
    z.object({ hookEventName: z.literal('Elicitation'), ... }),
    z.object({ hookEventName: z.literal('ElicitationResult'), ... }),
    z.object({ hookEventName: z.literal('CwdChanged'), ... }),
    z.object({ hookEventName: z.literal('FileChanged'), ... }),
    z.object({ hookEventName: z.literal('WorktreeCreate'), ... }),
  ]).optional(),
})
```

### 5.1 通用字段

| 字段 | 含义 |
|---|---|
| `continue` | 是否继续主流程（false 中断） |
| `suppressOutput` | hook 的 stdout 不在 transcript 显示 |
| `stopReason` | continue=false 时的停止原因（给用户看） |
| `decision` | `'approve' \| 'block'`——某些事件能用这个粗粒度决策 |
| `reason` | decision 的解释 |
| `systemMessage` | hook 想发的警告消息 |

### 5.2 `hookSpecificOutput` 按事件特化（72-163）

每个 hook 事件的输出 **schema 不同**——因为不同事件能影响的东西不同。

#### `PreToolUse`（72-78）—— 最强的 hook

```ts
{
  hookEventName: 'PreToolUse',
  permissionDecision: permissionBehaviorSchema().optional(),  // allow/deny/ask
  permissionDecisionReason: z.string().optional(),
  updatedInput: z.record(z.string(), z.unknown()).optional(), // 改 tool input
  additionalContext: z.string().optional(),                   // 注入额外上下文
}
```

**4 个能力**：

1. `permissionDecision` — 直接决定 allow/deny/ask
2. `permissionDecisionReason` — 决策解释
3. `updatedInput` — 修改 tool input（拦截 path、改 args）
4. `additionalContext` — 给模型加额外提示

任何一个都能影响主流程。这就是为什么 PreToolUse hook 是最危险也最强的——hook 命令里写错一行可能让整个 conversation 走偏。

#### `UserPromptSubmit` / `SessionStart` / `Setup` / `SubagentStart` / `PostToolUse`（79-111）

```ts
{ hookEventName: 'UserPromptSubmit', additionalContext?: string }
{ hookEventName: 'SessionStart', additionalContext?: string, initialUserMessage?: string, watchPaths?: string[] }
{ hookEventName: 'Setup', additionalContext?: string }
{ hookEventName: 'SubagentStart', additionalContext?: string }
{ hookEventName: 'PostToolUse', additionalContext?: string, updatedMCPToolOutput?: unknown }
{ hookEventName: 'PostToolUseFailure', additionalContext?: string }
```

**注入 context 是这些事件的核心能力**——hook 给模型加一段 prompt。比如 `UserPromptSubmit` 自动加"今天 X 月 Y 日"。

`SessionStart.initialUserMessage` 能让 hook **替用户敲第一条消息**——自动开始任务。

`SessionStart.watchPaths` / `CwdChanged.watchPaths` / `FileChanged.watchPaths` —— 让 hook 注册 watched 路径，后续 `FileChanged` 事件按它过滤。

`PostToolUse.updatedMCPToolOutput` —— 改 MCP tool 的输出（让模型看到改过的）。

#### `PermissionRequest`（120-134）—— 弹对话框前的拦截

```ts
{
  hookEventName: 'PermissionRequest',
  decision: z.union([
    z.object({
      behavior: z.literal('allow'),
      updatedInput: ...,
      updatedPermissions: z.array(permissionUpdateSchema()).optional(),
    }),
    z.object({
      behavior: z.literal('deny'),
      message: z.string().optional(),
      interrupt: z.boolean().optional(),
    }),
  ]),
}
```

**hook 能在权限对话框弹出前直接 allow/deny**——让 hook 充当 permission engine 的扩展。

`updatedPermissions` 让 hook 同时**保存权限规则**（"以后总是允许"等价物）。

#### `PermissionDenied` / `Notification` / `Elicitation` / `ElicitationResult` 等更多事件

每个有自己的 schema——参考 113-162 行，这里不一一列。

### 5.3 这种"按事件 narrow schema"的设计

`hookSpecificOutput` 是 discriminated union，按 `hookEventName` 判别。每事件 schema 只暴露**这个事件能影响的字段**——TypeScript 帮你拒绝写错（"在 SessionEnd hook 里写 permissionDecision 类型就报错"）。

这种 **schema-as-API-contract** 是 zod 的标准模式，但用得很彻底。

---

## 6. 段 3：hookJSONOutputSchema — async 和 sync 联合（169-200）

```ts
export const hookJSONOutputSchema = lazySchema(() => {
  const asyncHookResponseSchema = z.object({
    async: z.literal(true),
    asyncTimeout: z.number().optional(),
  })
  return z.union([asyncHookResponseSchema, syncHookResponseSchema()])
})
```

#### Hook 可以同步 / 异步响应

- **Sync** —— hook 命令立即完成，stdout 是完整 syncHookResponseSchema JSON
- **Async** —— hook 命令立即返回 `{async: true, asyncTimeout?: N}`，告诉系统"我会在另一渠道（attachment）异步回复"。系统继续主流程，等真正结果时通过 attachment 处理

`isSyncHookJSONOutput` / `isAsyncHookJSONOutput` 是类型 guard（182-193）：

```ts
export function isSyncHookJSONOutput(json: HookJSONOutput): json is SyncHookJSONOutput {
  return !('async' in json && json.async === true)
}
```

这种 union 让 hook 命令既可以快返回（同步），也可以"我要慢，请别等我"（异步）。**两种模式一个 schema 表达**。

### 6.1 编译期 SDK type matching 断言

行 196-200 有个**编译期类型断言**：

```ts
import type { IsEqual } from 'type-fest'
type Assert<T extends true> = T
type _assertSDKTypesMatch = Assert<
  IsEqual<SchemaHookJSONOutput, HookJSONOutput>
>
```

意思：**zod schema 推导出的 TS 类型 必须 等于 SDK 类型**——如果某天不等，TS 编译报错。

这是**保护类型一致性**的方法——zod 是 runtime validator，SDK type 是 client-facing 类型，两者不一致会导致客户端验证通过但类型不对（或反之）。这一行编译期断言强制它们对齐。

非常优雅的做法。

---

## 7. 段 4：HookCallbackContext / HookCallback（202-232）

```ts
export type HookCallbackContext = {
  getAppState: () => AppState
  updateAttributionState: (updater: (prev: AttributionState) => AttributionState) => void
}

export type HookCallback = {
  type: 'callback'
  callback: (
    input: HookInput,
    toolUseID: string | null,
    abort: AbortSignal | undefined,
    hookIndex?: number,
    context?: HookCallbackContext,
  ) => Promise<HookJSONOutput>
  timeout?: number
  internal?: boolean
}

export type HookCallbackMatcher = {
  matcher?: string
  hooks: HookCallback[]
  pluginName?: string
}
```

### 7.1 程序化 hook 的形态

`HookCallback` 是**代码定义的 hook**——区别于 settings.json 里的 shell command 形式。

```ts
type: 'callback'
```

判别符——区分声明式 settings 来的（type: 'command'）和程序化的（type: 'callback'）。

`callback` 函数签名暴露：

- `input: HookInput` — 事件特定的 input
- `toolUseID: string | null` — 关联到 tool 调用（null = 非 tool 事件）
- `abort: AbortSignal | undefined` — 取消信号
- `hookIndex?: number` — SessionStart hooks 用（注释 218："compute CLAUDE_ENV_FILE path"）
- `context?: HookCallbackContext` — 拿 AppState 等

返回 `Promise<HookJSONOutput>`——和 shell command hook 一样的 schema 输出。**程序化 hook 和声明式 hook 走同一套输出协议**——这是同 model 表达的关键。

### 7.2 `internal: boolean` 标志

注释 224-225：

```
Internal hooks (e.g. session file access analytics) are excluded from tengu_run_hook metrics
```

**内部 hook 不计入用户 hook telemetry**——避免 Claude Code 自己的 hook 把"用户写了多少 hook"统计搞乱。

### 7.3 HookCallbackMatcher

```ts
export type HookCallbackMatcher = {
  matcher?: string
  hooks: HookCallback[]
  pluginName?: string
}
```

**一组 hook 的容器**——matcher 模式 + 多个 callbacks + 可选 pluginName（哪个 plugin 注册的）。

settings.json 的 hooks 配置反序列化后大致就是 `Map<HookEvent, HookCallbackMatcher[]>`：

```ts
{
  PreToolUse: [
    { matcher: 'Bash(git:*)', hooks: [callback1, callback2] },
    { matcher: 'Read', hooks: [callback3] },
  ],
  PostToolUse: [
    { hooks: [callback4] },         // 无 matcher = 匹配所有
  ],
}
```

### 7.4 `matcher` 的语法

string——按事件类型解释。常见：

- `'Bash(git:*)'` — 匹配特定 tool + sub-pattern
- `'*'` 或省略 — 匹配所有
- `'mcp__github__*'` — MCP tool 前缀

具体语法由 hook 引擎和每个 tool 的 `preparePermissionMatcher` 共同决定。

---

## 8. 段 5：HookProgress / HookBlockingError（234-246）

```ts
export type HookProgress = {
  type: 'hook_progress'
  hookEvent: HookEvent
  hookName: string
  command: string
  promptText?: string
  statusMessage?: string
}

export type HookBlockingError = {
  blockingError: string
  command: string
}
```

### 8.1 HookProgress 是中间状态上报

hook 命令在执行中可以多次产生 progress——比如长跑 hook 报"已处理 X 条"。这种 progress 类型写入 ProgressMessage 里（03 篇 7 节）。

注意 `type: 'hook_progress'` 这个判别符——和 ToolProgress 同位置同模式。

### 8.2 HookBlockingError 是 blocking 失败的载体

hook 报告"我阻止了主流程"时附带 error message + 是哪个 command 阻止的。让 UI 能展示出来。

---

## 9. 段 6：PermissionRequestResult（248-258）

```ts
export type PermissionRequestResult =
  | {
      behavior: 'allow'
      updatedInput?: Record<string, unknown>
      updatedPermissions?: PermissionUpdate[]
    }
  | {
      behavior: 'deny'
      message?: string
      interrupt?: boolean
    }
```

PermissionRequest hook 的决策 schema。和 `PermissionAllowDecision` / `PermissionDenyDecision` 类似但简化版——hook 只需要 allow/deny，不需要 ask（hook 是替代 ask 的）。

`interrupt?: boolean` 在 deny 里——让 hook 不只 deny 单次调用，还**打断整个 turn**。这是个核灵——比 deny 更狠的拒绝。

---

## 10. 段 7：HookResult / AggregatedHookResult（260-290）

### 10.1 HookResult — 单个 hook 跑完的结果

```ts
export type HookResult = {
  message?: Message
  systemMessage?: Message
  blockingError?: HookBlockingError
  outcome: 'success' | 'blocking' | 'non_blocking_error' | 'cancelled'
  preventContinuation?: boolean
  stopReason?: string
  permissionBehavior?: 'ask' | 'deny' | 'allow' | 'passthrough'
  hookPermissionDecisionReason?: string
  additionalContext?: string
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}
```

#### 4 种 outcome

```
'success'              - 正常成功
'blocking'             - 阻止了主流程（阻塞）
'non_blocking_error'   - 报错但不阻止
'cancelled'            - 被取消
```

#### 各种字段对应不同 hookSpecificOutput 的产出

- `additionalContext` ← UserPromptSubmit / SessionStart / 等
- `initialUserMessage` ← SessionStart
- `updatedInput` ← PreToolUse
- `updatedMCPToolOutput` ← PostToolUse
- `permissionRequestResult` ← PermissionRequest
- `retry` ← PermissionDenied

**HookResult 是把所有事件输出"扁平化"的中间表征**——把 hookSpecificOutput 的特化字段平铺到 HookResult 的 optional 字段，让消费方处理简单。

### 10.2 AggregatedHookResult（277-290）

```ts
export type AggregatedHookResult = {
  message?: Message
  blockingErrors?: HookBlockingError[]
  preventContinuation?: boolean
  stopReason?: string
  hookPermissionDecisionReason?: string
  permissionBehavior?: PermissionResult['behavior']
  additionalContexts?: string[]                  // ← 注意复数
  initialUserMessage?: string
  updatedInput?: Record<string, unknown>
  updatedMCPToolOutput?: unknown
  permissionRequestResult?: PermissionRequestResult
  retry?: boolean
}
```

#### 多个 hook 的结果合并

- `blockingErrors: []` — 多个 hook 都阻塞，全收集
- `additionalContexts: []` — 多个 hook 都注入 context，全合并
- 其它单值字段 — 取**最后一个非空**或**优先级最高**

这种"单结果 → 聚合结果"的两层设计反映**hook 调用是多对一**——一个事件可能多个 hook 注册，全跑完才汇总。

#### 同名字段语义不同

- `HookResult.additionalContext: string` (单数)
- `AggregatedHookResult.additionalContexts: string[]` (复数)

字段名复数让消费方一眼看出"这是聚合的"——避免"我以为是单个 context 但其实是多个拼起来"的隐患。

---

## 11. 整套 Hook 在 model 网里的关系

```
                  用户在 settings.json 写
                  ┌──────────────────────────┐
                  │ {                         │
                  │   hooks: {                │
                  │     PreToolUse: [         │
                  │       {                   │
                  │         matcher: '*',     │
                  │         hooks: [          │
                  │           { type: 'command',│
                  │             command: '...'}│
                  │         ]                 │
                  │       }                   │
                  │     ]                     │
                  │   }                       │
                  │ }                         │
                  └──────────┬───────────────┘
                             │ load + parse
                             ▼
                  ┌──────────────────────────┐
                  │  HooksSettings            │
                  │  (utils/settings/types)   │
                  └──────────┬───────────────┘
                             │ register
                             ▼
                  ┌──────────────────────────┐
                  │  AppState.sessionHooks   │
                  │  Map<event, HookCallback │
                  │           Matcher[]>     │
                  └──────────┬───────────────┘
                             │
                  事件触发：tool 即将跑
                             │
                             ▼
                  ┌──────────────────────────┐
                  │  匹配 PreToolUse hooks    │
                  │  按 matcher 过滤          │
                  └──────────┬───────────────┘
                             │ 跑每个 hook
                             ▼
                  ┌──────────────────────────┐
                  │  HookCallback.callback    │
                  │  (HookInput, toolUseID,   │
                  │   abort, idx, context) →  │
                  │  HookJSONOutput           │
                  └──────────┬───────────────┘
                             │ 多 hook 输出
                             ▼
                  ┌──────────────────────────┐
                  │  汇总成                   │
                  │  AggregatedHookResult    │
                  └──────────┬───────────────┘
                             │
              影响什么：
              ├─→ permissionBehavior → 替代权限决策
              ├─→ updatedInput → 改 tool input
              ├─→ additionalContexts → 注入 prompt
              ├─→ blockingErrors → 阻止 tool 跑
              ├─→ updatedMCPToolOutput → 改 tool 输出
              └─→ message / systemMessage → 注入 conversation
```

---

## 12. 这一篇你应该带走的几样东西

1. 27 种 HookEvent 按"触发时机"分 7 类：tool / session / agent / compact / 用户 / 通知 / 配置
2. PreToolUse 是最强的 hook（4 个能力：决策 / 改 input / 改 reason / 注入 context）
3. `hookSpecificOutput` 是 discriminated union——每事件 schema 只暴露能影响的字段
4. async hook（`{async: true}`）vs sync hook 用 union 表达，type guard 区分
5. 编译期 zod schema vs SDK type 一致性断言（`Assert<IsEqual<...>>`）的优雅设计
6. HookCallback 的 `type: 'callback'` 区分程序化 vs 声明式
7. HookResult vs AggregatedHookResult —— 单 hook vs 多 hook 聚合，复数字段名提示
8. PermissionRequest hook 的 `interrupt: boolean` 比普通 deny 更狠（中断整个 turn）
9. `internal: true` 让内部 hook 不污染用户 telemetry
10. matcher 的语法由各 tool 的 preparePermissionMatcher 解释——Bash 是 glob，Skill 是前缀

---

## 13. 设计哲学要点

1. **声明式 + 程序化双轨**：用户写 shell command，代码注册 callback——同一套输出协议
2. **事件细粒度暴露**：27 种事件让用户精确订阅，避免粗粒度+if-else
3. **schema-as-API-contract**：每事件的特化字段用 zod literal 判别，编译期就拒绝错配
4. **结果两层（单 + 聚合）**：单 hook 结果按事件特化，多 hook 聚合后扁平化
5. **async/sync 一个 union**：`{async: true}` 让慢 hook 不阻塞主流程
6. **类型一致性硬保证**：`Assert<IsEqual<SchemaType, SDKType>>` 在编译期断言
7. **内部 / 外部分离**：`internal: true` 让系统 hook 不污染用户 metrics
8. **复数字段名**：`additionalContext` vs `additionalContexts` 让聚合语义自显
9. **hook 充当权限引擎扩展**：PreToolUse 和 PermissionRequest 都能直接决策 allow/deny

下一篇 07 看 `Plugin` —— hook 通过 plugin 来源被装载，plugin 是 hook 的来源之一。
