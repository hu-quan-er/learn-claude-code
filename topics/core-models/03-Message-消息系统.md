# 03. Message — 消息系统

> 这一篇做一件事：把 Claude Code 的消息体系作为 model 讲透——`UserMessage` / `AssistantMessage` / `SystemMessage`（含 14 种 subtype）/ `AttachmentMessage`（含 30+ 种 attachment）/ `ProgressMessage` / `TombstoneMessage` 这 6 大类。
>
> **重要前置说明**：`restored-src/src/types/message.ts` 在快照里**缺失**——这个文件被 `utils/messages.ts:41-73` 等多处 `import type` 引用，但实际类型定义不在快照里。本篇所有 model 描述基于：
>
> 1. `utils/messages.ts` 的**构造函数**（`createUserMessage`、`createAssistantMessage` 等共 23 个）
> 2. `utils/attachments.ts` 的 **`Attachment` union 定义**（440-... 行，这部分在快照里）
> 3. tool 实现里对 message 字段的**实际读取**
>
> 反推出来的 model **结构上准确**（构造函数返回的对象必然符合类型），但**字段精确签名**可能有偏差（只读字段 / 可选性）。读这一篇时把"反推"当作 lemma，遇到关键决策再去构造函数本身确认。

---

## 1. Message 在 model 网里的位置

Claude Code 的 conversation 是一个 `Message[]` 数组。所有：

- 用户输入
- 模型输出
- tool 调用与结果
- 系统通知（错误、压缩、API 状态）
- 附件（文件引用、@-mention、skill listing）
- 进度上报

都是 `Message`。这个数组：

- 持久化到磁盘（session log）
- 部分序列化发给 Anthropic API
- 全部用于 UI 渲染
- 用于 conversation recovery（resume）
- 用于 compact（压缩总结）

也就是说 Message **既是渲染数据**，**又是 API 请求载荷**，**又是持久化记录**——三种用途同时承担。

这逼出了 Claude Code 消息 model 的几个关键设计：

- 一种 message 不只发 API（`isMeta`、`isVisibleInTranscriptOnly` 等可见性 flag）
- 不所有 message 都进 transcript（`isVirtual`）
- 系统消息和模型消息要清晰分开
- attachment 要建模成"消息"，因为它要按时间序混进 conversation

---

## 2. 顶层联合：6 大类

按 `type` 字段判别（基于 `utils/messages.ts:41-73` 的 import 列表）：

```ts
type Message =
  | UserMessage             // type: 'user'
  | AssistantMessage        // type: 'assistant'
  | SystemMessage           // type: 'system'
  | AttachmentMessage       // type: 'attachment'
  | ProgressMessage         // type: 'progress'
  | TombstoneMessage        // type: 'tombstone'
```

每一类下面展开。

---

## 3. UserMessage — 用户/系统视角的输入

### 3.1 字段（从 `createUserMessage` 反推，行 460-525）

```ts
type UserMessage = {
  type: 'user'
  message: {
    role: 'user'
    content: string | ContentBlockParam[]    // 文本 或 SDK 消息块数组
  }
  uuid: UUID
  timestamp: string                            // ISO 8601
  
  // 可见性 flags（决定 message 是否进 API / 是否进 transcript）
  isMeta?: true                                // 隐藏于用户但模型可见
  isVisibleInTranscriptOnly?: true            // 只展示，不发 API
  isVirtual?: true                            // 不进 transcript（后台占位）
  isCompactSummary?: true                     // compact 产生的摘要
  
  // 业务相关
  toolUseResult?: unknown                      // 如果这是 tool_result，原 tool 的输出
  mcpMeta?: { _meta?: ...; structuredContent?: ... }
  imagePasteIds?: number[]
  sourceToolAssistantUUID?: UUID              // 关联到产生此 tool_result 的 assistant message
  permissionMode?: PermissionMode             // 发送时的权限模式（rewind 用）
  summarizeMetadata?: {                        // compact 摘要附加信息
    messagesSummarized: number
    userContext?: string
    direction?: PartialCompactDirection
  }
  origin?: MessageOrigin                       // 来源（人输入 / 系统注入 / 桥接 / ...）
}
```

### 3.2 三个可见性 flag — 这是 Claude Code 消息系统的关键设计

| flag | 模型可见 | 用户可见（transcript）| 典型场景 |
|---|---|---|---|
| 默认（无 flag） | ✓ | ✓ | 用户敲的话 |
| `isMeta: true` | ✓ | **隐藏** | skill 注入的内容、命令 metadata、attachment 文本 |
| `isVisibleInTranscriptOnly: true` | **不**（API 前过滤） | ✓ | 本地命令 stdout、UI 装饰 |
| `isVirtual: true` | （视情况） | **不进 transcript** | 后台占位、临时状态 |

**这是非常精细的可见性建模**——一个消息的"展示"和"发送给 API"是两个独立的决策。

为什么需要这种区分？因为 Claude Code 既是 LLM 客户端（要发完整 prompt），又是 CLI（要展示给用户），还是 transcript log（要记录历史）。三方对"哪些消息该看"有冲突。每个 flag 解决一种"我在哪个视图里出现 / 不出现"的问题。

#### `isMeta` 的典型用法

skill 注入（`processSlashCommand.tsx:909` 那条 user message）就是 `isMeta: true`——用户敲了 `/verify`，UI 不会再次显示一大段 SKILL_BODY，但模型一定要看到。

### 3.3 `toolUseResult` 和 `sourceToolAssistantUUID`

这两个一起出现——表示**这条 user message 实际是 tool 的执行结果**，应该被关联回触发它的 assistant message（含 tool_use）。

设计上 tool result 用 user message 体现，是因为 Anthropic API 的 messages 数组只能交替 user/assistant role——tool_result 必须以 user 角色发回。

`sourceToolAssistantUUID` 让 UI 知道"这条 tool_result 是哪个 tool_use 的回应"，能渲染成嵌套的"tool 调用 + 结果"块。

### 3.4 `MessageOrigin`

来自 `types/message.ts`（缺失），但从 `utils/messages.ts` 可见用法。它标识消息来源：

- 人键盘输入
- 系统注入
- bridge（remote control）
- replay
- ...

用在 telemetry 和 UI 标签——比如远程 bridge 来的消息显示一个图标。

---

## 4. AssistantMessage — 模型回复

### 4.1 字段（从 `baseCreateAssistantMessage` 反推，行 355-409）

```ts
type AssistantMessage = {
  type: 'assistant'
  uuid: UUID
  timestamp: string
  message: {                                   // BetaMessage from Anthropic SDK
    id: string                                 // model 给的 message id (msg_xxx)
    container: null | ContainerInfo
    model: string                              // 实际用的模型
    role: 'assistant'
    stop_reason: 'stop_sequence' | 'end_turn' | 'tool_use' | 'max_tokens' | ...
    stop_sequence: string
    type: 'message'
    usage: {                                   // 完整 token 用量
      input_tokens: number
      output_tokens: number
      cache_creation_input_tokens: number
      cache_read_input_tokens: number
      server_tool_use: { web_search_requests, web_fetch_requests }
      service_tier: string | null
      cache_creation: { ephemeral_1h_input_tokens, ephemeral_5m_input_tokens }
      inference_geo: string | null
      iterations: number | null
      speed: number | null
    }
    content: BetaContentBlock[]                // text / thinking / redacted_thinking / tool_use 块
    context_management: null | unknown
  }
  requestId: string | undefined                // API request_id (req_xxx)
  
  // 错误处理
  apiError?: { ... }
  error?: SDKAssistantMessageError
  errorDetails?: string
  isApiErrorMessage: boolean
  
  // 可见性
  isVirtual?: true
}
```

### 4.2 设计上几个有意思的点

#### `message.id` 和 `requestId` 都保留

- `message.id` (`msg_xxx`) — Anthropic 给的消息 ID
- `requestId` (`req_xxx`) — HTTP 请求 ID

两个都有用：

- `msg_xxx` 用于 message-level 操作（编辑、引用）
- `req_xxx` 用于和 server-side 日志关联（性能分析、cache miss 排查）

不丢任何一个。

#### `usage` 是丰富的而非"input + output 两个数字"

完整暴露：

- 普通 input/output token
- cache 命中读 / cache creation
- server-side tool（web search、web fetch）的请求数
- service tier（standard / batch）
- cache 创建时长（5m vs 1h）
- 推理地理位置
- iterations、speed（speed 是 streaming 速度估算）

这些字段都被 telemetry 和 cost-tracker 消费——精细化计费、性能优化、cache 调优都需要。

#### `apiError` vs `error` 的拆分

- `apiError` — Anthropic SDK 抛的 APIError
- `error` — SDK 包装的 `SDKAssistantMessageError`（更结构化）

两套并存说明历史上经历过 schema 变更。

#### `isVirtual` 在 assistant message 也存在

模型没真发 API（fake assistant message——比如 compact 后的合成响应、初始 placeholder）也可以表达成 AssistantMessage，但 `isVirtual: true`。

---

## 5. SystemMessage — 14 种 subtype

System message 是 UI 和系统状态消息——**不发给 API**（注意区别于 Claude API 的 `system` prompt，这两个不是一回事）。

按 `subtype` 判别。从 `utils/messages.ts` grep 得到的全部 14 种：

| `subtype` | 用途 | 创建函数 |
|---|---|---|
| `informational` | 通用信息（warning、info） | `createSystemMessage` |
| `permission_retry` | "已允许 X，重试" | `createPermissionRetryMessage` |
| `bridge_status` | bridge 连接状态 | `createBridgeStatusMessage` |
| `scheduled_task_fire` | scheduled task 触发 | `createScheduledTaskFireMessage` |
| `stop_hook_summary` | stop hook 多个结果摘要 | `createStopHookSummaryMessage` |
| `turn_duration` | turn 耗时 | `createTurnDurationMessage` |
| `away_summary` | away 模式总结 | `createAwaySummaryMessage` |
| `memory_saved` | 记忆已保存通知 | `createMemorySavedMessage` |
| `agents_killed` | agent 被终止 | `createAgentsKilledMessage` |
| `api_metrics` | API 调用指标 | `createApiMetricsMessage` |
| `local_command` | 本地命令 stdout | `createCommandInputMessage` |
| `compact_boundary` | 压缩边界 | `createCompactBoundaryMessage` |
| `microcompact_boundary` | 微压缩边界 | `createMicrocompactBoundaryMessage` |
| `api_error` | API 错误 | `createSystemAPIErrorMessage` |

### 5.1 公共字段（从所有 create 函数反推）

```ts
type SystemMessageBase = {
  type: 'system'
  subtype: SystemMessageSubtype           // 上表 14 种之一
  content: string
  isMeta: boolean                         // 大多 false（用户可见）
  timestamp: string
  uuid: UUID
  toolUseID?: string                      // 关联到某次 tool 调用
  level?: SystemMessageLevel              // 'info' | 'warning' | 'error'
  preventContinuation?: boolean           // 阻止主 query 继续
}
```

每个 subtype 在公共字段之外加自己的 schema。例如：

```ts
SystemBridgeStatusMessage = SystemMessageBase & {
  subtype: 'bridge_status'
  url: string
  upgradeNudge?: string
}

SystemPermissionRetryMessage = SystemMessageBase & {
  subtype: 'permission_retry'
  commands: string[]                      // 哪些命令被允许了
}

SystemCompactBoundaryMessage = SystemMessageBase & {
  subtype: 'compact_boundary'
  // ... 含 messagesSummarized, summarized totals 等
}
```

### 5.2 `appendSystemMessage`（Tool.ts:206-209）的类型约束

```ts
appendSystemMessage?: (
  msg: Exclude<SystemMessage, SystemLocalCommandMessage>,
) => void
```

注意 `Exclude<SystemMessage, SystemLocalCommandMessage>`——**`local_command` 这一种系统消息禁止通过这个钩子注入**。

为什么？`local_command` 是用户敲了 slash 命令产生的 stdout，必须由 `processSlashCommand` 流程统一产生（保证 UI 顺序、metadata 一致）。Tool 的 `appendSystemMessage` 是给"我想给用户喊一句话"用的——不能伪造一条用户命令输出。

**类型层面就阻止了误用**。这种"用类型表达约束"非常成熟。

### 5.3 `level: SystemMessageLevel`

```ts
type SystemMessageLevel = 'info' | 'warning' | 'error'
```

UI 渲染颜色 / 图标依据。注意它是字符串枚举不是数字——后者是某些遗留代码喜欢用的"语义可读"模式。

---

## 6. AttachmentMessage — 容纳 30+ 种 attachment

### 6.1 顶层结构

```ts
type AttachmentMessage = {
  type: 'attachment'
  attachment: Attachment       // ← 30+ 种 union
  uuid: UUID
  timestamp: string
}
```

`createAttachmentMessage`（`attachments.ts:3201-3210`）很简单：

```ts
export function createAttachmentMessage(attachment: Attachment): AttachmentMessage {
  return {
    attachment,
    type: 'attachment',
    uuid: randomUUID(),
    timestamp: new Date().toISOString(),
  }
}
```

attachment message **不存 content**——content 隐藏在 `attachment` 里。这让"附件"和"文本"分开建模。

### 6.2 `Attachment` union（`attachments.ts:440-...`）

总共 **30+ 种** attachment 类型。按主题分组：

#### 文件相关

| `type` | 含义 |
|---|---|
| `file` | @-mention 文件 |
| `compact_file_reference` | compact 后的文件引用占位 |
| `pdf_reference` | PDF 引用 |
| `already_read_file` | 已读过的文件（去重提示） |
| `edited_text_file` | 文件被编辑了 |
| `edited_image_file` | 图片文件被编辑 |
| `directory` | 目录列表 |
| `selected_lines_in_ide` | IDE 选中行 |
| `opened_file_in_ide` | IDE 打开文件 |

#### 记忆相关

| `type` | 含义 |
|---|---|
| `nested_memory` | 嵌套 CLAUDE.md |
| `relevant_memories` | 相关记忆（user/feedback/project/reference） |

#### Skill 相关

| `type` | 含义 |
|---|---|
| `dynamic_skill` | 动态发现的 skill 提示 |
| `skill_listing` | skill 列表（增量推送） |
| `skill_discovery` | skill 搜索结果（实验性） |

#### 命令相关

| `type` | 含义 |
|---|---|
| `command_permissions` | skill 临时工具权限 |
| `queued_command` | 排队的命令 |

#### Agent / 任务相关

| `type` | 含义 |
|---|---|
| `agent_mention` | @-mention 一个 agent |
| `todo_reminder` | TODO 提醒 |
| `task_reminder` | task 提醒 |

#### Mode 相关

| `type` | 含义 |
|---|---|
| `output_style` | 输出风格 |
| `plan_mode` | 计划模式提示 |
| `plan_mode_reentry` | 重入计划模式 |
| `plan_mode_exit` | 退出计划模式 |
| `auto_mode` | auto 模式提示 |
| `auto_mode_exit` | 退出 auto 模式 |

#### Hook / 系统相关

| `type` | 含义 |
|---|---|
| `async_hook_response` | 异步 hook 响应 |
| `hook_blocking_error` | hook 阻塞错误 |
| `hook_stopped_continuation` | hook 阻止继续 |
| `hook_additional_context` | hook 注入额外上下文 |
| `hook_permission_decision` | hook 权限决策 |
| `hook_system_message` | hook 系统消息 |
| `hook_cancelled` | hook 取消 |
| `hook_error_during_execution` | hook 执行错误 |
| `hook_success` | hook 成功 |
| `hook_non_blocking_error` | hook 非阻塞错误 |

#### 其它

| `type` | 含义 |
|---|---|
| `diagnostics` | LSP 诊断信息 |
| `critical_system_reminder` | 严重系统提示 |

### 6.3 为什么把 attachment 建模成独立 message 类型而不是混进 user message content？

#### 第一个原因：生命周期

很多 attachment 是**临时的**——比如 `todo_reminder` 只在某个 turn 注入一次，下个 turn 不再发。如果混进 user message，user message 是持久的，attachment 怎么"过期"？建模成独立 message + 合适的 lifecycle 处理（compact 时丢弃某些 attachment、永久保留某些）让生命周期清晰。

#### 第二个原因：渲染独立

attachment 的渲染和普通 user message 不同——todo_reminder 渲染成一个折叠卡片，skill_listing 渲染成跳过卡片，nested_memory 渲染成"加载了 CLAUDE.md"提示。每种 attachment 有自己的 renderer——做独立 message 类型让 UI 路由清晰。

#### 第三个原因：API 序列化

attachment **不一定**直接发给 API。`skill_listing` 会被序列化成一段 system reminder 文本；`hook_*` 部分不发 API（只是 UI 用）。如果混进 user message content 强制发 API，要么过滤复杂，要么发了垃圾。

### 6.4 一个特别值得看的：`relevant_memories`

```ts
| {
    type: 'relevant_memories'
    memories: {
      path: string
      content: string
      mtimeMs: number
      header?: string                       // ★ 预计算！
      limit?: number
    }[]
  }
```

注释 506-516（`attachments.ts`）：

```
Pre-computed header string (age + path prefix). Computed once
at attachment-creation time so the rendered bytes are stable
across turns — recomputing memoryAge(mtimeMs) at render time
calls Date.now(), so "saved 3 days ago" becomes "saved 4 days
ago" across turns → different bytes → prompt cache bust.
```

**这是 prompt cache 优化的极致案例**：渲染层如果调 `Date.now()` 算"X 天前"，**字节会随时间漂移**，prompt cache 失效。所以 header 在 attachment 创建时就**冻结**好，渲染层只读不算。

这种"冻结渲染输出"的细节反映了 Claude Code 对 prompt cache 命中率的执着追求——任何会让 prompt 字节漂移的源头都被处理。

---

## 7. ProgressMessage — 异步进度

### 7.1 字段（行 603-619）

```ts
type ProgressMessage<P extends Progress = Progress> = {
  type: 'progress'
  data: P                                    // 由具体 tool 决定
  toolUseID: string                          // 关联 tool_use
  parentToolUseID: string                    // 嵌套场景（agent in agent）
  uuid: UUID
  timestamp: string
}
```

### 7.2 用途

tool 在执行过程中可以多次回调 `onProgress`——产生 ProgressMessage。例子：

- BashProgress: stdout 流式数据
- AgentToolProgress: 子 agent 内部消息
- WebSearchProgress: "已查询 N 条"
- HookProgress: hook 执行中

ProgressMessage **不发 API**（看 Tool.ts:312-319 的 `filterToolProgressMessages`），只用于 UI。

### 7.3 `parentToolUseID` 和 `toolUseID` 都存

子 agent 跑 tool 时，progress 同时关联到子 tool（`toolUseID`）和父调用（`parentToolUseID`）。这让 UI 能渲染成"父 → 子 → 进度"的嵌套树。

---

## 8. TombstoneMessage — 已删除/被替换的消息占位

具体定义没在快照里见到，但从 import 列表（`utils/messages.ts:70`）可以推断它是**消息被删除后的墓碑**——保留 UUID 让别人引用不会断掉，但内容被丢掉。

典型场景：rewind（回滚）时，被回滚到某个点之前的某些消息可能被替换为 tombstone。

---

## 9. NormalizedMessage — API 边界的转换

`utils/messages.ts:46-48` 又导入了三个：

```ts
NormalizedAssistantMessage
NormalizedMessage
NormalizedUserMessage
```

这是**发给 API 之前**的归一化形态。区别于运行时形态：

| 运行时 Message | NormalizedMessage |
|---|---|
| 含 `isMeta` / `isVirtual` 等 flag | 已剥离（API 不需要） |
| 含 `uuid` / `timestamp` 等元数据 | 部分保留 |
| `content` 可能是 `string` 或 `ContentBlockParam[]` | 强制 `ContentBlockParam[]` |
| 可能含 attachment-derived content（tool_result 关联） | tool_result 已正确放回 user content |

转换发生在 `normalizeMessagesForAPI`（grep 之能找到）——把内部表征翻译成 API 期望的格式。

**这种"内部 model + 边界归一化"模式是经典的领域驱动设计**——内部为业务建模舒服，边界处转换成接口需要的形状。

---

## 10. Message 在所有 model 里的位置

```
                  ┌───────────────────────┐
                  │  Message[] (历史)      │
                  │  conversation 数组    │
                  └────────┬──────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
   持久化到磁盘         发送给 API         UI 渲染
   (session log)     (after normalize)   (transcript)
        │                  │                  │
        │ 通过 isVirtual   │ 通过 isMeta /    │ 通过 subtype /
        │ 部分跳过         │ isVisibleInTrans │ attachment.type
        │                  │ scriptOnly       │ 路由 renderer
        │                  │ 过滤             │
        ▼                  ▼                  ▼
   conversation       Anthropic SDK      React 组件
   recovery 用        BetaMessage 数组    每种 type/subtype
                                          一个 renderer
                                          
        │
        │ 各类 message 都引用其它 model：
        │
        ├──→ Tool.ts ToolUseContext.messages 字段
        │    （tool 执行时能看历史）
        │
        ├──→ AppState.tasks[*].messages
        │    （每个 task 有自己的 message 数组）
        │
        ├──→ Hook 系统注入 attachmentMessage
        │    (HookCallbackContext / HookResult)
        │
        ├──→ ToolResult.newMessages
        │    （tool 完成后注入新消息）
        │
        └──→ Command.getPromptForCommand 返回
             ContentBlockParam[]
             被包成 user message (isMeta:true)
```

Message 是 model 网里**关联面最大的**——所有 model 都通过它来"和 conversation 对话"。

---

## 11. 这一篇你应该带走的几样东西

读到这里，你应该能：

1. 解释 Message 顶层 6 大类按 `type` 字段判别的设计
2. 区分 UserMessage 的 4 个可见性 flag（`isMeta` / `isVisibleInTranscriptOnly` / `isVirtual` / `isCompactSummary`）
3. 解释为什么 tool_result 用 user message 表达
4. 列出 SystemMessage 14 种 subtype 的至少 6 种
5. 解释为什么 `appendSystemMessage` 用 `Exclude<SystemMessage, SystemLocalCommandMessage>`
6. 说出 AttachmentMessage 至少 5 个分组主题（文件 / 记忆 / skill / hook / mode 等）
7. 解释为什么 attachment 不混进 user message content
8. `relevant_memories.header` 的"预计算 + 冻结"是为了 prompt cache 命中率
9. ProgressMessage 不发 API、只用于 UI
10. NormalizedMessage 是 API 边界的转换形态

---

## 12. 设计哲学要点

1. **可见性是独立维度**：`isMeta` / `isVisibleInTranscriptOnly` / `isVirtual` 三个 flag 解决了"我对谁可见"——单一字段建模不够的问题。
2. **建模历史而非状态**：消息是 append-only 数组，不是 mutable state——简化了并发、recovery、UI re-render。
3. **API 边界转换**：内部 model 复杂、API 简洁——`normalizeMessagesForAPI` 桥接两者。
4. **每种特殊用途一个 type/subtype**：14 种 SystemMessage subtype、30+ 种 Attachment——不强行抽象成"通用消息"。
5. **Prompt cache 是头号公民**：`relevant_memories.header` 这种"渲染输出冻结"的细节为 cache 让步。
6. **关联性显式建模**：`sourceToolAssistantUUID`、`toolUseID`、`parentToolUseID`、`requestId` 各自表达一种关联，不混用。

---

## 13. 缺口提醒

本篇基于反推。**正式工作时如果碰到字段疑问**：

- 类型签名查 `utils/messages.ts` 的 `import type` 列表 + 各 `create*` 构造函数
- attachment 字段查 `utils/attachments.ts:440-...` 的 union 定义
- 序列化逻辑查 `normalizeMessagesForAPI` (grep)
- 渲染查 `components/messages/*.tsx`

下一篇 04 看 `AppState` —— 全局可变状态 model。
