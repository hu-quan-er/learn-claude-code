# 11 - SDK 编程接口与会话管理

---

## 一、SDK 架构概述

### 1.1 核心文件职责

| 文件 | 职责 |
|------|------|
| `src/entrypoints/agentSdkTypes.ts` | 主 SDK API 导出和类型定义（444 行） |
| `src/entrypoints/sdk/coreSchemas.ts` | Zod Schema 定义（1890+ 行）— 所有 SDK 数据类型 |
| `src/entrypoints/sdk/controlSchemas.ts` | 控制协议 Schema — 初始化、中断 |
| `src/entrypoints/sdk/coreTypes.ts` | 从 Schema 生成的 TypeScript 类型 |
| `src/remote/sdkMessageAdapter.ts` | CCR WebSocket 消息到内部 REPL 类型的转换 |

### 1.2 SDK 公共 API

```typescript
// 会话管理（v2 不稳定 API）
unstable_v2_createSession(options: SDKSessionOptions): SDKSession
unstable_v2_resumeSession(sessionId: string, options: SDKSessionOptions): SDKSession
unstable_v2_prompt(message: string, options: SDKSessionOptions): Promise<SDKResultMessage>

// 会话内省
getSessionMessages(sessionId: string, options?): Promise<SessionMessage[]>
listSessions(options?): Promise<SDKSessionInfo[]>
getSessionInfo(sessionId: string, options?): Promise<SDKSessionInfo | undefined>

// 会话变更
renameSession(sessionId: string, title: string, options?): Promise<void>
tagSession(sessionId: string, tag: string | null, options?): Promise<void>
forkSession(sessionId: string, options?): Promise<ForkSessionResult>

// 守护进程原语（内部）
watchScheduledTasks(): AsyncIterable<ScheduledTaskEvent>
connectRemoteControl(): RemoteControlHandle
```

---

## 二、SDK 消息类型体系

### 2.1 消息联合类型

```typescript
SDKMessage =
  | SDKAssistantMessage          // 模型响应
  | SDKUserMessage               // 用户输入
  | SDKUserMessageReplay         // 用户消息重放
  | SDKResultMessage             // 会话完成（成功/错误）
  | SDKSystemMessage             // 初始化消息
  | SDKPartialAssistantMessage   // 流式部分响应
  | SDKCompactBoundaryMessage    // 压缩边界标记
  | SDKStatusMessage             // 状态更新
  | SDKAPIRetryMessage           // API 重试通知
  | SDKLocalCommandOutputMessage // 本地命令输出
  | SDKHookStartedMessage        // 钩子启动
  | SDKHookProgressMessage       // 钩子进度
  | SDKHookResponseMessage       // 钩子响应
  | SDKToolProgressMessage       // 工具进度
  | SDKAuthStatusMessage         // 认证状态
  | SDKTaskNotificationMessage   // 任务通知
  | SDKTaskStartedMessage        // 任务启动
  | SDKTaskProgressMessage       // 任务进度
  | SDKSessionStateChangedMessage // 会话状态变更
  | SDKFilesPersistedEvent       // 文件持久化事件
  | SDKToolUseSummaryMessage     // 工具使用摘要
  | SDKRateLimitEvent            // 速率限制事件
  | SDKElicitationCompleteMessage // 引出完成
  | SDKPromptSuggestionMessage   // 提示建议
```

### 2.2 核心消息结构

**SDKUserMessage**（用户输入）：
```typescript
{
  type: 'user'
  message: APIUserMessage
  parent_tool_use_id: string | null
  isSynthetic?: boolean
  tool_use_result?: unknown
  priority?: 'now' | 'next' | 'later'
  timestamp?: ISO string
  uuid?: UUID
  session_id?: string
}
```

**SDKAssistantMessage**（模型响应）：
```typescript
{
  type: 'assistant'
  message: APIAssistantMessage
  parent_tool_use_id: string | null
  error?: 'authentication_failed' | 'billing_error' | 'rate_limit' | ...
  uuid: UUID
  session_id: string
}
```

**SDKResultMessage**（会话完成）：
```typescript
{
  type: 'result'
  subtype: 'success' | 'error_during_execution' | 'error_max_turns' | ...
  duration_ms: number
  duration_api_ms: number
  num_turns: number
  total_cost_usd: number
  usage: ModelUsage
  modelUsage: Record<string, ModelUsage>
  permission_denials: SDKPermissionDenial[]
  // 成功时：
  result: string
  stop_reason: string
  // 错误时：
  errors: string[]
  uuid: UUID
  session_id: string
}
```

---

## 三、SDK 钩子事件系统

### 3.1 25 种钩子事件类型

```typescript
HookEventType =
  // 工具生命周期
  | 'PreToolUse' | 'PostToolUse' | 'PostToolUseFailure'
  // 通知
  | 'Notification'
  // 用户交互
  | 'UserPromptSubmit'
  // 会话生命周期
  | 'SessionStart' | 'SessionEnd' | 'Stop' | 'StopFailure'
  // 子代理
  | 'SubagentStart' | 'SubagentStop'
  // 压缩
  | 'PreCompact' | 'PostCompact'
  // 权限
  | 'PermissionRequest' | 'PermissionDenied'
  // 系统
  | 'Setup' | 'TeammateIdle'
  // 任务
  | 'TaskCreated' | 'TaskCompleted'
  // 引出
  | 'Elicitation' | 'ElicitationResult'
  // 配置与文件
  | 'ConfigChange' | 'WorktreeCreate' | 'WorktreeRemove'
  | 'InstructionsLoaded' | 'CwdChanged' | 'FileChanged'
```

### 3.2 钩子基础输入

```typescript
// 所有钩子共享的基础输入
{
  session_id: string
  transcript_path: string
  cwd: string
  permission_mode?: string
  agent_id?: string         // 子代理钩子
  agent_type?: string       // --agent 或子代理上下文
}
```

---

## 四、SDK 会话生命周期

### 4.1 创建流程

```
1. unstable_v2_createSession(options) 调用
   → options 包含: model, cwd, instructions, agents, mcp, hooks 等

2. SDK 通过 --sdk-url 桥接传输生成 CLI 子进程

3. CLI 通过 SDKControlInitializeRequest 初始化

4. 响应返回：可用命令、模型、账户信息、fast_mode_state

5. 生成并返回会话 ID
```

### 4.2 消息流传生命周期

```
1. SDK 发送 SDKUserMessage（或 AsyncIterable<SDKUserMessage>）

2. CLI 处理：运行工具、调用钩子、生成助手响应

3. 产出 SDK 消息序列：
   → 钩子进度（SDKHookStartedMessage, SDKHookProgressMessage）
   → 工具进度（SDKToolProgressMessage）
   → 部分响应（SDKPartialAssistantMessage + stream_event）
   → 完整助手消息（SDKAssistantMessage）
   → 状态更新（SDKStatusMessage, SDKSessionStateChangedMessage）

4. 最终产出 SDKResultMessage
```

### 4.3 恢复流程

```
1. unstable_v2_resumeSession(sessionId, options) 调用

2. CLI 从 ~/.claude/projects/<sanitized-path>/<sessionId>.jsonl 加载会话

3. 通过 parentUuid 链接构建对话链（buildConversationChain）

4. 从元数据加载：
   → 文件历史、归属、内容替换
   → Todo 列表、工作树状态
   → PR 链接、自定义标题、标签

5. 产出 SDKSystemMessage（包含初始化状态）

6. 准备接收新的 SDKUserMessage
```

---

## 五、会话存储架构

### 5.1 核心文件职责

| 文件 | 职责 |
|------|------|
| `src/utils/sessionStorage.ts` | 主会话持久化逻辑（2900+ 行）— Project 单例类 |
| `src/utils/sessionStoragePortable.ts` | 可移植工具（350+ 行）— UUID 验证、元数据读取 |
| `src/commands/resume/resume.tsx` | /resume 命令 UI — 会话浏览与选择 |
| `src/utils/sessionRestore.ts` | 状态重建 — 文件历史、Todo、工作树恢复 |
| `src/assistant/sessionHistory.ts` | 远程历史分页（88 行）— CCR 分页 |
| `src/history.ts` | 本地命令历史（150 行）— 粘贴文本引用 |

### 5.2 JSONL 条目类型

```typescript
type Entry =
  | TranscriptMessage              // 用户/助手/附件/系统消息
  | SummaryMessage                 // AI 生成的摘要
  | CustomTitleMessage             // /rename 持久化
  | AiTitleMessage                 // AI 自动标题
  | LastPromptMessage              // 最近的用户提示
  | TaskSummaryMessage             // Fork agent 中期状态
  | TagMessage                     // /tag 持久化
  | AgentNameMessage               // Swarm 团队成员名称
  | AgentColorMessage              // Swarm 团队成员颜色
  | AgentSettingMessage            // --agent 类型
  | PRLinkMessage                  // GitHub PR 链接
  | FileHistorySnapshotMessage     // Undo 历史快照
  | AttributionSnapshotMessage     // 提交归属状态
  | QueueOperationMessage          // 消息队列操作
  | SpeculationAcceptMessage       // 推测性执行
  | ModeEntry                      // 'coordinator' | 'normal'
  | WorktreeStateEntry             // 工作树进入/退出状态
  | ContentReplacementEntry        // 替换的内容存根
  | ContextCollapseCommitEntry     // 归档的消息组
  | ContextCollapseSnapshotEntry   // 暂存队列 + 触发器
```

### 5.3 TranscriptMessage 结构

```typescript
type TranscriptMessage = SerializedMessage & {
  parentUuid: UUID | null               // 对话链
  logicalParentUuid?: UUID | null       // parentUuid 被置空时保留
  isSidechain: boolean                  // Agent 子代理转录
  gitBranch?: string
  agentId?: string                      // 侧链关联用
  teamName?: string
  agentName?: string
  agentColor?: string
  promptId?: string                     // OTel 关联 ID
}
```

---

## 六、Project 单例类

### 6.1 核心属性

```typescript
class Project {
  // 当前会话元数据缓存（最后写入获胜）
  currentSessionTag?: string
  currentSessionTitle?: string
  currentSessionAgentName?: string
  currentSessionAgentColor?: string
  currentSessionLastPrompt?: string
  currentSessionAgentSetting?: string
  currentSessionMode?: 'coordinator' | 'normal'
  currentSessionWorktree?: PersistedWorktreeSession | null
  currentSessionPrNumber?: number
  currentSessionPrUrl?: string
  currentSessionPrRepository?: string

  sessionFile: string | null              // JSONL 文件路径
  private pendingEntries: Entry[] = []    // 缓冲到 materializeSessionFile
  private remoteIngressUrl: string | null
  private internalEventWriter?: (event, payload) => Promise<void>
  private writeQueues: Map<string, QueuedEntry[]>
  private flushTimer: NodeJS.Timeout | null
  private activeDrain: Promise<void> | null
  private FLUSH_INTERVAL_MS = 100        // 批处理刷新间隔
  private readonly MAX_CHUNK_BYTES = 100 * 1024 * 1024  // 100MB
}
```

### 6.2 写入批处理机制

```
1. enqueueWrite() → 按文件路径入队

2. scheduleDrain() → 如果没有待处理定时器，设置 100ms 定时器

3. drainWriteQueue() → 将条目批处理到 MAX_CHUNK_BYTES
   → 追加写入文件系统
   → 多个块可能用于大量刷新（100MB+ 文件）
   → 所有 resolver 在块写入后调用

4. 进程退出时 Project.flush() 确保所有排队写入完成
```

---

## 七、会话持久化生命周期

### 7.1 创建与首次写入

```
1. 初始状态
   → 会话 ID 生成（UUID v4，crypto.randomUUID）
   → Project 创建，sessionFile = null
   → 条目缓冲在 pendingEntries[]

2. 首次消息（materializeSessionFile）
   → 收到第一条用户/助手消息时，解析 sessionFile 路径
   → 路径: ~/.claude/projects/<sanitized-cwd>/<sessionId>.jsonl
   → pendingEntries 刷新到磁盘
   → sessionFile = getTranscriptPath()
```

### 7.2 正常追加流程

```
appendEntry(entry):
  ├── 元数据条目（custom-title, tag, last-prompt）→ 始终入队
  ├── 转录消息 → 去重检查
  │   ├── 新 UUID → 加入 messageSet + 入队
  │   └── 重复 UUID → 跳过
  ├── 侧链（agentId 已设置）→ 写入 Agent 专用文件
  └── 远程持久化（CCR 已连接）→ 异步通过 sessionIngress 追加
```

### 7.3 远程持久化（Session Ingress HTTP API）

```
追加流程：
1. appendEntry() → persistToRemote()（如果 CCR 已连接）
2. getSessionIngressAuthToken() 获取 JWT
3. 按 sessionId 创建顺序包装器（防止竞态）
4. HTTP PUT 到远程 ingress（含 Last-Uuid 头）
5. 409 冲突时：采用服务器的 UUID，重试

重试策略：
  MAX_RETRIES = 10
  指数退避: 500ms → 1000ms → 2000ms → ... → 8000ms 上限
  401 立即失败（认证错误）
  5xx/429/网络错误 → 重试
  409 → 获取服务器日志发现 lastUuid

CCR v2 替代方案（进程内事件）：
  setInternalEventWriter() 注册回调
  appendEntry() 调用 writer('transcript', entry, {isCompaction?, agentId?})
  完全避免 HTTP
```

### 7.4 会话清理

```
registerCleanup 处理器：
1. 进程退出时调用 Project.flush()
2. 确保所有排队写入完成
3. reAppendSessionMetadata() 在刷新后调用
   → 从文件尾部读取以吸收 SDK 外部写入
   → custom-title 和 tag 被 SDK 覆盖 → CLI 可见
   → 将元数据重新追加到 EOF 保持在尾部读取窗口中
```

---

## 八、会话恢复机制

### 8.1 会话发现

```
listSessions / getSessionInfo：
1. 扫描 ~/.claude/projects/ 目录
2. 每个文件：readSessionLite()
   → 读取头部（前 65KB）+ 尾部（后 65KB）
   → 从头部提取 firstPrompt
   → 从尾部提取 customTitle/tag/agent-*
3. 返回轻量 LogOption[]（不加载完整消息）
```

### 8.2 LogOption 元数据结构

```typescript
type LogOption = {
  date: string                          // "2024-01-15"
  messages: SerializedMessage[]         // 完整转录（如果加载）
  fullPath?: string
  created: Date
  modified: Date
  firstPrompt: string                   // 提取的首条用户提示
  messageCount: number
  fileSize?: number                     // JSONL 文件大小
  isSidechain: boolean
  isLite?: boolean                      // 轻量加载仅元数据
  sessionId?: string
  teamName?: string                     // Swarm 团队成员名称
  agentName?: string
  customTitle?: string                  // 用户设置的标题（/rename）
  tag?: string                          // 用户标签（/tag）
  gitBranch?: string
  projectPath?: string
  prNumber?: number                     // 关联的 GitHub PR
  prUrl?: string
  mode?: 'coordinator' | 'normal'
  worktreeSession?: PersistedWorktreeSession | null
  summary?: string                      // AI 生成的摘要
  leafUuid?: UUID                       // 最后消息 UUID
  // ... 更多字段
}
```

### 8.3 对话链重建

```
buildConversationChain：
1. 从 JSONL 加载所有消息
2. 跳过非转录条目（元数据、文件历史等）
3. 通过 parentUuid → message 链接构建对话树
4. 处理遗留进度条目（跨越桥接）
5. 处理会话断点（logicalParentUuid 保留）
6. 按时间顺序排列
```

### 8.4 状态恢复

```
restoreSessionStateFromLog：
1. 文件历史快照 → 重建 Undo 状态
2. 归属快照 → 恢复提交归属跟踪
3. 最后 TodoWrite 工具使用块 → 恢复 Todo
4. WorktreeStateEntry → 恢复工作树会话
5. PR 链接、Agent 元数据、模式
```

---

## 九、会话 Fork

```
forkSession 流程：
1. 加载源会话的 JSONL
2. 创建新会话 ID
3. 重映射所有消息 UUID（每条消息新 ID）
4. 保留 parentUuid 链（调整为新 UUID）
5. 可选 upToMessageId 截断（从特定点分支）
6. 写入新会话 JSONL
7. 文件历史快照不复制（全新 Undo 状态）
8. 返回 ForkSessionResult { sessionId }
```

---

## 十、远程会话历史

### 10.1 分页 API

```typescript
const HISTORY_PAGE_SIZE = 100  // 每页事件数

// 获取最新页
fetchLatestEvents(sessionId): Promise<SDKMessage[]>
  → anchor_to_latest 参数获取最新页

// 获取更早事件
fetchOlderEvents(sessionId, beforeId): Promise<SDKMessage[]>
  → before_id 参数向前分页

// 认证
→ OAuth 头 + 组织 UUID（CCR 认证）
```

---

## 十一、跨系统集成

### 11.1 SDK → JSONL 持久化

```
SDK 产出 SDKMessage 类型
  → 适配器转换为内部 Message
  → 序列化为 TranscriptMessage
  → 追加到 JSONL
  → 如果 CCR 已连接，同时通过 sessionIngress 发送
```

### 11.2 钩子与会话

```
钩子输入包含 session_id 和 transcript_path
  → 钩子可读取 transcript_path 检查对话
  → SessionStart/SessionEnd 钩子在恢复时触发
  → 恢复时 source='resume'
```

### 11.3 子代理协调

```
每个子代理获得独立的 agent-<agentId>.jsonl 文件
  → 父会话的 sessionStorage 跟踪 Agent 转录子目录
  → 恢复时，Agent 转录补充到 AgentTool 状态
  → Agent 元数据（.meta.json 副车）持久化 Agent 类型 + 工作树路径
```

### 11.4 定时任务

```
存储在 <project>/.claude/scheduled_tasks.json
  → 非按会话持久化（项目内所有会话共享）
  → 守护进程通过 watchScheduledTasks() 监控
  → PID 活性检查防止跨 REPL + 守护进程双重触发
```

---

## 十二、实现模式总结

| 模式 | 说明 |
|------|------|
| **写入批处理** | 按文件路径队列，100ms 刷新间隔，100MB 最大块 |
| **轻量 vs 完整加载** | readSessionLite() 仅读 65KB 头+尾 vs 完整 JSONL |
| **UUID 去重** | Set\<UUID\> 跟踪已写入消息，防止恢复或并发写入时重复 |
| **按会话顺序化** | sessionIngress.ts 使用 sequential() 包装器防止远程追加竞态 |
| **尾部元数据刷新** | reAppendSessionMetadata() 吸收 SDK 外部写入保持一致性 |
| **延迟加载** | loadTranscriptFile() 可限制加载大小（MAX_TRANSCRIPT_READ_BYTES = 50MB） |
| **Lazy Schema** | 所有 Zod schema 包装在 lazySchema() 避免循环导入 |
| **乐观并发** | Last-Uuid 头实现远程持久化的乐观并发控制 |
