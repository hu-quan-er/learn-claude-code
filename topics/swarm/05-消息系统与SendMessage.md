# 05 消息系统与 SendMessage

> teammate 之间怎么通信——file-based mailbox (1183 行) + SendMessage 工具 (917 行)。本篇拆 inbox 文件结构、lockfile 并发、SendMessage 的 3 类目标（teammate / broadcast / cross-session）、消息自动送达机制。

## 5.1 file-based mailbox 全貌

每个 teammate 在团队目录里有一个 inbox 文件：

```
~/.claude/teams/{teamName}/
└── inboxes/
    ├── team-lead.json
    ├── researcher.json
    └── coder.json
```

inbox 文件是 **JSON 数组**，每个元素是 `TeammateMessage`：

```ts
// teammateMailbox.ts:41-49
export type TeammateMessage = {
  from: string          // 发送者 agent name
  text: string          // 消息文本（可能是 JSON.stringify 的协议消息）
  timestamp: string     // ISO 时间戳
  read: boolean         // 收件人读过没
  color?: string        // 发送者 UI 颜色
  summary?: string      // 5-10 字摘要，UI 预览用
}
```

注意 **`read: boolean`** 字段——mailbox 不只是 queue，而是带 read 状态的持久化邮箱。读了的消息不立刻删，标记为 read 留作历史。

## 5.2 inbox 路径生成

```ts
// teammateMailbox.ts:56-66
export function getInboxPath(agentName: string, teamName?: string): string {
  const team = teamName || getTeamName() || 'default'
  const safeTeam = sanitizePathComponent(team)
  const safeAgentName = sanitizePathComponent(agentName)
  const inboxDir = join(getTeamsDir(), safeTeam, 'inboxes')
  const fullPath = join(inboxDir, `${safeAgentName}.json`)
  return fullPath
}
```

两层 sanitization：team name 和 agent name 都过 `sanitizePathComponent`——防止路径注入（`../evil.json` 这种）。

`teamName` 优先用参数 > `getTeamName()` 返回值 > `'default'` 兜底。`'default'` fallback 让 swarm 关闭时 mailbox 仍能工作（虽然意义不大）。

## 5.3 lockfile 并发控制

`teammateMailbox.ts:29-37`：

```ts
const LOCK_OPTIONS = {
  retries: {
    retries: 10,
    minTimeout: 5,
    maxTimeout: 100,
  },
}
```

`proper-lockfile` 的配置：

- 最多重试 10 次；
- 每次重试间隔 5-100 ms（指数退避）；
- 总等待时间约 ~500ms 上限。

为什么用 lockfile？因为多个 teammate 可能**并发**写同一个 inbox（如多个 worker 同时给 lead 发消息）。如果不加锁：

1. Worker A 读 inbox.json，看到 3 条消息；
2. Worker B 读 inbox.json，看到同样 3 条；
3. Worker A 追加自己的消息写回（4 条）；
4. Worker B 追加自己的消息写回（4 条，覆盖 A 写的）；
5. A 的消息丢了。

lockfile 强制串行：每个 writer 必须先获得 `inbox.json.lock` 文件的独占锁才能读写。

注释（`teammateMailbox.ts:26-29`）：

> Lock options: retry with backoff so concurrent callers (multiple Claudes in a swarm) wait for the lock instead of failing immediately. The sync lockSync API blocked the event loop; the async API needs explicit retries to achieve the same serialization semantics.

明确说明：**async lockfile API 必须显式 retry** 才能达到 sync 版本的语义。这是个 Node.js 异步 IO 常见陷阱——sync API 阻塞但能保证串行，async API 不阻塞但 caller 要自己处理"已被锁就等"的情况。

## 5.4 writeToMailbox 完整流程

`teammateMailbox.ts:134-192`：

```ts
export async function writeToMailbox(
  recipientName: string,
  message: Omit<TeammateMessage, 'read'>,
  teamName?: string,
): Promise<void> {
  await ensureInboxDir(teamName)

  const inboxPath = getInboxPath(recipientName, teamName)
  const lockFilePath = `${inboxPath}.lock`

  // Step 1: 确保 inbox 文件存在 (proper-lockfile 要求)
  try {
    await writeFile(inboxPath, '[]', { encoding: 'utf-8', flag: 'wx' })
  } catch (error) {
    const code = getErrnoCode(error)
    if (code !== 'EEXIST') {
      logError(error)
      return
    }
  }

  let release: (() => Promise<void>) | undefined
  try {
    // Step 2: 获取锁 (带重试)
    release = await lockfile.lock(inboxPath, {
      lockfilePath: lockFilePath,
      ...LOCK_OPTIONS,
    })

    // Step 3: 锁定后重新读取最新状态
    const messages = await readMailbox(recipientName, teamName)

    // Step 4: 追加新消息
    const newMessage: TeammateMessage = { ...message, read: false }
    messages.push(newMessage)

    // Step 5: 写回
    await writeFile(inboxPath, jsonStringify(messages, null, 2), 'utf-8')
  } catch (error) {
    logError(error)
  } finally {
    // Step 6: 释放锁
    if (release) await release()
  }
}
```

6 步流程的几个细节：

### Step 1：`flag: 'wx'` —— 仅在文件不存在时创建

```ts
await writeFile(inboxPath, '[]', { encoding: 'utf-8', flag: 'wx' })
```

`wx` 模式：write + 排他（exclusive）创建。如果文件已存在 → 抛 `EEXIST`。catch 里识别 `EEXIST` 视为正常（文件已存在不需要重新创建），其它错误才上报。

注释明确："proper-lockfile requires the file to exist"——锁的对象是文件，没文件没法锁。

### Step 3：**锁定后重新读取**

```ts
release = await lockfile.lock(...)
// Re-read messages after acquiring lock to get the latest state
const messages = await readMailbox(recipientName, teamName)
```

获取锁**之后**才读 inbox——而不是锁之前。这是关键的并发正确性：

- 锁之前读：可能拿到旧状态，被另一个 writer 改过；
- 锁之后读：拿到的就是当前最新状态。

这是经典的 **read-after-lock** 模式。

### Step 6：`finally` 释放

锁必须在 `finally` 里释放——即使写失败也要释放，否则 inbox 死锁。

## 5.5 readMailbox / readUnreadMessages —— 读取不加锁

注意**读取不加锁**：

```ts
// teammateMailbox.ts:84-108
export async function readMailbox(agentName: string, teamName?: string): Promise<TeammateMessage[]> {
  const inboxPath = getInboxPath(agentName, teamName)
  try {
    const content = await readFile(inboxPath, 'utf-8')
    return jsonParse(content) as TeammateMessage[]
  } catch (error) {
    const code = getErrnoCode(error)
    if (code === 'ENOENT') return []
    return []
  }
}
```

直接读、不锁。为什么？

- 读不修改文件，**多个 reader 并发是安全的**；
- 加锁会增加延迟、且某些场景下（如 UI 渲染）希望 best-effort 读；
- 即使读到一个"正在被 write" 的中间状态——`writeFile` 在 POSIX 上是原子的（write + rename），所以读不会看到撕裂的 JSON。

但是有个**理论 race**：reader 看到旧状态，write 完成后 reader 不会再看。实际不影响——下一轮 query loop 会重新读。

## 5.6 mark as read 机制

`teammateMailbox.ts:201-271` 的 `markMessageAsReadByIndex`：

```ts
export async function markMessageAsReadByIndex(
  agentName: string,
  teamName: string | undefined,
  messageIndex: number,
): Promise<void> {
  // ... 加锁 ...
  const messages = await readMailbox(agentName, teamName)

  if (messageIndex < 0 || messageIndex >= messages.length) return

  const message = messages[messageIndex]
  if (!message || message.read) return  // 已读或不存在 → noop

  messages[messageIndex] = { ...message, read: true }
  await writeFile(inboxPath, jsonStringify(messages, null, 2), 'utf-8')
  // ... 释放锁 ...
}
```

为什么需要 mark as read？因为 messages-pipeline 在每轮 query loop 都注入未读消息（`readUnreadMessages`）。模型处理完后必须标记 read，否则下一轮还会注入同样的消息——**死循环重复 attention**。

`markMessagesAsRead`（不带 Index 的版本）一次性标记所有未读为 read。

## 5.7 SendMessage 工具的输入 schema

`SendMessageTool.ts:67-87`：

```ts
const inputSchema = lazySchema(() =>
  z.object({
    to: z
      .string()
      .describe(
        feature('UDS_INBOX')
          ? 'Recipient: teammate name, "*" for broadcast, "uds:<socket-path>" for a local peer, or "bridge:<session-id>" for a Remote Control peer (use ListPeers to discover)'
          : 'Recipient: teammate name, or "*" for broadcast to all teammates',
      ),
    summary: z.string().optional().describe('...'),
    message: z.union([
      z.string().describe('Plain text message content'),
      StructuredMessage(),  // 见 5.8
    ]),
  }),
)
```

3 个字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `to` | string | 收件人地址 |
| `summary` | string? | 5-10 字 UI 预览 |
| `message` | string \| StructuredMessage | 消息内容 |

### `to` 字段的 4 种地址

| 格式 | 含义 |
|------|------|
| `"researcher"` | 团队内 teammate 名字 |
| `"*"` | 广播给所有 teammate |
| `"uds:/tmp/xxx.sock"` | 同机器 Claude session 的 Unix socket（gated `UDS_INBOX`） |
| `"bridge:session_abc"` | 跨机器 Remote Control peer（gated `UDS_INBOX`） |

后两种是**跨 session 通信**（不限于 swarm 内）。Gate `UDS_INBOX` 控制是否启用。`SendMessageTool/prompt.ts` 的工具描述会按 gate 显示对应字段说明。

### message —— string 或 StructuredMessage

`SendMessageTool.ts:46-65`：

```ts
const StructuredMessage = lazySchema(() =>
  z.discriminatedUnion('type', [
    z.object({
      type: z.literal('shutdown_request'),
      reason: z.string().optional(),
    }),
    z.object({
      type: z.literal('shutdown_response'),
      request_id: z.string(),
      approve: semanticBoolean(),
      reason: z.string().optional(),
    }),
    z.object({
      type: z.literal('plan_approval_response'),
      request_id: z.string(),
      approve: semanticBoolean(),
      feedback: z.string().optional(),
    }),
  ]),
)
```

3 种结构化消息：
- `shutdown_request`：要求 teammate 关闭；
- `shutdown_response`：teammate 回复是否同意关闭；
- `plan_approval_response`：teammate 回复是否同意 plan。

注意：**模型可以直接传 JSON 对象作为 `message`**，Zod 解析后区分是 string 还是 structured。详细的协议状态机在 [06](./06-协议消息状态机.md) 展开。

## 5.8 SendMessage 工具的 prompt 教育

回顾 `SendMessageTool/prompt.ts`（[00 总览](./00-总览与代码地图.md) 引用过）的关键句：

> Your plain text output is NOT visible to other agents — to communicate, you MUST call this tool. Messages from teammates are delivered automatically; you don't check an inbox. Refer to teammates by name, never by UUID. When relaying, don't quote the original — it's already rendered to the user.

3 条关键教育：

### 1. plain text 不可见

模型在对话流里说的话**只用户能看到**——teammate 看不到。**必须**用 SendMessage 工具才能跨 teammate 传递信息。

这避免模型误以为"我在对话流里 @ 一下 researcher 它就能看到"——明确告诉模型唯一的通信通道是 SendMessage。

### 2. 消息自动送达

> Messages from teammates are delivered automatically; **you don't check an inbox**.

模型**不需要主动**查 inbox——messages-pipeline 的 attachment surfacer（[messages-pipeline 07](../messages-pipeline/07-attachment-normalizer.md) 提过）会自动注入未读消息。

这降低模型的认知负担——它只管发消息，接收是自动的。

### 3. relaying 不引用原文

> When relaying, **don't quote the original** — it's already rendered to the user.

如果 teammate 给 lead 发了消息，lead 在向用户报告时**不要复述消息内容**——因为 UI 已经把 teammate 消息渲染给用户看过了。复述会让用户看到两遍。

这种"教育模型避免特定 anti-pattern" 是 prompt engineering 的标志。

## 5.9 formatTeammateMessages —— XML 包装

`teammateMailbox.ts:373-389`：

```ts
export function formatTeammateMessages(messages): string {
  return messages
    .map(m => {
      const colorAttr = m.color ? ` color="${m.color}"` : ''
      const summaryAttr = m.summary ? ` summary="${m.summary}"` : ''
      return `<${TEAMMATE_MESSAGE_TAG} teammate_id="${m.from}"${colorAttr}${summaryAttr}>\n${m.text}\n</${TEAMMATE_MESSAGE_TAG}>`
    })
    .join('\n\n')
}
```

输出 XML 格式：

```
<teammate-message teammate_id="researcher" color="red" summary="found 3 bugs">
Found 3 bugs in auth.ts...
</teammate-message>

<teammate-message teammate_id="coder" color="blue">
Started on task #2
</teammate-message>
```

这是消息**注入到模型上下文**的格式——通过 `<teammate-message>` 标签让模型清楚地区分"这段是 teammate 说的"。

这个 XML 标签和 [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md) 的 `<system-reminder>` 是同一类设计——**用标签让模型识别消息来源**。

## 5.10 IdleNotification —— teammate 每个 turn 结束自动报告

`teammateMailbox.ts:394-447` 定义了 idle notification 机制：

```ts
export type IdleNotificationMessage = {
  type: 'idle_notification'
  from: string
  timestamp: string
  idleReason?: 'available' | 'interrupted' | 'failed'
  summary?: string
  completedTaskId?: string
  completedStatus?: 'resolved' | 'blocked' | 'failed'
  failureReason?: string
}
```

teammate 完成一个 turn 后**自动**给 lead 发 idle_notification（通过 Stop hook）：

- `idleReason`: 怎么 idle 的（正常完成 / 被打断 / 失败）；
- `summary`: 本 turn 的 5-10 字摘要；
- `completedTaskId` / `completedStatus`: 如果完成了任务，哪个、什么状态。

这让 lead **被动**知道 teammate 状态——不需要 lead 主动问。

`isIdleNotification(messageText)` 检查消息是否是 idle_notification 类型：

```ts
export function isIdleNotification(messageText: string): IdleNotificationMessage | null {
  try {
    const parsed = jsonParse(messageText)
    if (parsed && parsed.type === 'idle_notification') {
      return parsed as IdleNotificationMessage
    }
  } catch {}
  return null
}
```

UI 用这个判定来**特殊渲染**——idle_notification 不像普通消息那样显示完整内容，而是用一个简短的"researcher: idle" 提示。

## 5.11 PermissionRequest/Response —— Worker → Leader 桥

`teammateMailbox.ts:450-520+` 定义了 worker 把权限请求发给 leader 的消息格式：

```ts
export type PermissionRequestMessage = {
  type: 'permission_request'
  request_id: string
  agent_id: string
  tool_name: string
  tool_use_id: string
  description: string
  input: Record<string, unknown>
  permission_suggestions: unknown[]
}

export type PermissionResponseMessage =
  | { type: 'permission_response'; request_id: string; subtype: 'success'; response?: {...} }
  | { type: 'permission_response'; request_id: string; subtype: 'error'; error: string }
```

这是 [08 Permission Sync](./08-Permission-Sync.md) 的核心数据结构。注意**字段名 snake_case**：

> Field names align with SDK `can_use_tool` (snake_case).

故意和 SDK 对齐——便于 worker 把请求转发给 SDK 控制层时不需要重命名。

这是个**协议层考虑**：worker 把 mailbox 收到的 PermissionResponse 直接传给 SDK 的 `can_use_tool` 回调，字段名一致 = 零开销转换。

## 5.12 8 种结构化消息类型

完整盘点 `teammateMailbox.ts` 定义的协议消息类型：

| 消息类型 | 方向 | 用途 |
|----------|------|------|
| `idle_notification` | worker → leader | teammate 完成 turn 报告 |
| `permission_request` | worker → leader | 请求权限 |
| `permission_response` | leader → worker | 权限决策回复 |
| `sandbox_permission_request` | worker → leader | sandbox 权限请求 |
| `sandbox_permission_response` | leader → worker | sandbox 决策 |
| `shutdown_request` | leader → worker | 请求关闭 |
| `shutdown_approved` / `shutdown_rejected` | worker → leader | 关闭决策 |
| `plan_approval_request` | worker → leader | 请求 plan 批准 |
| `plan_approval_response` | leader → worker | plan 决策 |

注意分布：
- **5 种 worker → leader**：idle_notification, permission_request, sandbox_permission_request, shutdown_approved/rejected, plan_approval_request；
- **3 种 leader → worker**：permission_response, sandbox_permission_response, shutdown_request, plan_approval_response。

**Leader 是协调者**——大部分决策由 leader 做出。Worker 只在自己想关闭 / 想做 plan 时主动发起。

详细的状态机在 [06 协议消息状态机](./06-协议消息状态机.md)。

## 5.13 写消息 → 自动送达的完整链路

把 5.1-5.12 串起来——一次"researcher 给 lead 发消息" 的完整路径：

```
1. researcher 的模型生成 tool_use:
   { name: 'SendMessage',
     input: { to: 'team-lead', summary: 'found bug', message: '...' } }
                  │
                  ▼
2. SendMessageTool.call():
   - 解析 to: 'team-lead' → teammate by name
   - parsed.message: string (不是 structured)
   - 调用 writeToMailbox('team-lead', message, teamName)
                  │
                  ▼
3. writeToMailbox:
   - getInboxPath → ~/.claude/teams/{team}/inboxes/team-lead.json
   - 确保文件存在 (wx mode)
   - 获取 lockfile (重试 10x)
   - 读取最新 messages
   - append 新消息: { from: 'researcher', text: '...', timestamp, read: false, color, summary }
   - 写回 inboxes/team-lead.json
   - 释放 lock
                  │
                  ▼
4. tool_result 返回 researcher: { success: true, ... }
                  │
                  ▼
   ─── researcher 的工作结束 ───
   ─── 切换视角到 team-lead 的进程 ───
                  │
                  ▼
5. team-lead 在下一轮 query loop 准备 API 请求:
   attachment surfacer 扫描 inbox:
   - readUnreadMessages('team-lead', teamName)
   - 找到 1 条未读: { from: 'researcher', text: '...', summary: 'found bug' }
                  │
                  ▼
6. 生成 attachment: {
     type: 'teammate_mailbox',
     messages: [{ from, text, timestamp, color, summary }]
   }
                  │
                  ▼
7. messages-pipeline normalizeAttachmentForAPI:
   case 'teammate_mailbox':
     formatTeammateMessages([{...}]) →
     "<teammate-message teammate_id=\"researcher\" color=\"red\" summary=\"found bug\">
       ...
      </teammate-message>"
                  │
                  ▼
8. 包装成 user message + isMeta + SR wrap
                  │
                  ▼
9. team-lead 的模型看到下一 turn 输入里有:
   "<teammate-message teammate_id=\"researcher\">..."
                  │
                  ▼
10. team-lead 模型处理这个消息, 决定下一步行动
                  │
                  ▼
11. 处理完后, markMessagesAsRead('team-lead', teamName)
    所有未读 → 标记 read
    防止下一轮重复注入
```

**11 步完成消息自动送达**——researcher 调用 SendMessage 后只需要 ~50ms（写文件 + 锁），lead 在下一轮 query loop 就能看到。

## 5.14 几个隐性设计判断

### 1. file-based mailbox 而不是 in-memory queue

理由（重述 [00 总览](./00-总览与代码地图.md) 提过）：
- 跨进程（tmux backend 各 teammate 独立进程）；
- 跨终端（用户可能多 terminal attach）；
- 可 resume（重启后能拿到未读消息）；
- 不需要服务进程（没有 broker）；
- lockfile 是 standard 工具。

### 2. read 不加锁 / write 加锁

读不修改 → 多 reader 并发安全；写要 read-modify-write → 必须串行。这是标准 RWMutex 模式的简化。

### 3. write 走 read-after-lock

锁定**之后**再读最新状态——避免基于过期数据 modify。这是经典并发陷阱。

### 4. async lockfile 需要显式 retry

注释明确指出"sync API blocked event loop"——但 async 版本默认 fail immediately。需要 retries 配置才能达到 sync 语义。Node.js 异步 IO 的常见陷阱。

### 5. mark as read 防重复注入

消息读完必须标记 read——否则下一轮 attachment surfacer 又把同样消息注入。这是个**协议契约**：reader 必须显式 ack。

### 6. 自动送达 = lead 不用主动 poll

`<teammate-message>` 通过 attachment surfacer 自动出现在 lead 上下文里。Lead 模型不需要"调 ListMail 工具"——降低认知负担。

### 7. summary 字段服务 UI

5-10 字 summary 让 UI 能在不展开消息的情况下显示预览（如 spinner 边上"researcher: found bug"）。模型必须填这个字段（prompt 教育过）。

### 8. 协议消息嵌入文本字段

shutdown_request 等是 JSON 对象，**嵌入到 message.text 字符串**里（JSON.stringify）。这让 mailbox 只需要存 string——不用扩展 schema 支持每种协议消息。

代价：reader 要 try-parse 检测是不是结构化消息。但 isShutdownRequest / isPlanApprovalRequest 等 helper 函数已经封装好。

### 9. 字段命名对齐 SDK

`PermissionRequestMessage` 用 snake_case 故意对齐 SDK——零开销转发。这种**外部协议优先于内部约定**的设计在工业代码里很常见。

### 10. 8 种协议消息类型，5+3 leader/worker 角色分布

不是对称的——leader 是协调者，决策权大多在它手上。这种**非对称协议设计**反映 swarm 的中心化协调模型（不是完全去中心化的 actor 网格）。

## 5.15 小结

- 每个 teammate 有 inbox 文件：`~/.claude/teams/{team}/inboxes/{agent}.json`；
- `TeammateMessage` 字段 6 个，含 `read` 状态（持久化邮箱不是 queue）；
- writeToMailbox 走 lockfile 串行化、async retry 模拟 sync 语义；
- readMailbox 不加锁（POSIX write 原子性保证），write 走 read-after-lock；
- mark as read 防消息在下一轮被重复注入；
- SendMessage 工具 4 种地址（teammate / broadcast / uds / bridge）；
- message 字段 union：string 或 StructuredMessage 之一（3 种协议类型）；
- 自动送达：attachment surfacer 每轮扫 inbox → 注入 `<teammate-message>` XML → mark read；
- 8 种结构化协议消息：idle / permission / sandbox-permission / shutdown / plan_approval 各对儿；
- Leader 决策权大，5+3 非对称角色分布。

下一篇 → [06 协议消息状态机](./06-协议消息状态机.md)
