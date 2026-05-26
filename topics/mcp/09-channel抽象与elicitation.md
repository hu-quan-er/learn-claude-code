# 09 channel 抽象与 elicitation

> `channelNotification.ts` (316) + `channelPermissions.ts` (240) + `channelAllowlist.ts` (76) + `elicitationHandler.ts` (313) ≈ 945 行——把 MCP server 变成**外部输入通道**（Telegram/iMessage/Discord）和**结构化用户表单**两种 inbound 模式。

## 9.1 channel 是什么

```
传统 MCP server: tools/resources/prompts —— 模型 → server
channel server:  上面那些 + notifications/claude/channel —— server → 模型
```

**channel** 是个 MCP server 但**额外能 push 消息给模型**。设计驱动："让 Claude Code 在你不打开终端时通过 Telegram/iMessage 跟你对话"。

代码注释（channelNotification.ts:1-17）：

> A "channel" (Discord, Slack, SMS, etc.) is just an MCP server that:
>   - exposes tools for outbound messages (e.g. `send_message`) — standard MCP
>   - sends `notifications/claude/channel` notifications for inbound — this file
>
> The notification handler wraps the content in a `<channel>` tag and enqueues it. SleepTool polls hasCommandsInQueue() and wakes within 1s. The model sees where the message came from and decides which tool to reply with (the channel's MCP tool, SendUserMessage, or both).

`<channel>` 标签包内容、SleepTool 1s 轮询醒、模型自己决定用哪个工具回——和 [swarm 08 SendUserMessage](../swarm/08-SendUserMessage与SleepTool.md) 共享 SleepTool 机制。

## 9.2 两个 inbound 协议

### notifications/claude/channel —— 普通消息

```ts
// channelNotification.ts:37
export const ChannelMessageNotificationSchema = lazySchema(() =>
  z.object({
    method: z.literal('notifications/claude/channel'),
    params: z.object({
      content: z.string(),
      // Opaque passthrough — thread_id, user, whatever the channel wants the
      // model to see. Rendered as attributes on the <channel> tag.
      meta: z.record(z.string(), z.string()).optional(),
    }),
  }),
)
```

`meta` 是**完全 opaque 的字符串对象**——Telegram 塞 `chat_id`、Slack 塞 `thread_ts`、Discord 塞 `message_id`。模型看到 `<channel chat_id="123">` 知道**这是哪个对话上下文**。

### notifications/claude/channel/permission —— 结构化权限回复

```ts
// channelNotification.ts:62
export const CHANNEL_PERMISSION_METHOD =
  'notifications/claude/channel/permission'
export const ChannelPermissionNotificationSchema = lazySchema(() =>
  z.object({
    method: z.literal(CHANNEL_PERMISSION_METHOD),
    params: z.object({
      request_id: z.string(),
      behavior: z.enum(['allow', 'deny']),
    }),
  }),
)
```

注释（channelNotification.ts:49-61）：

> Structured permission reply from a channel server. Servers that support this declare `capabilities.experimental['claude/channel/permission']` and emit this event INSTEAD of relaying "yes tbxkq" as text via notifications/claude/channel.
>
> The server parses the user's reply (spec: `/^\s*(y|yes|n|no)\s+([a-km-z]{5})\s*$/i`) and emits {request_id, behavior}. CC matches request_id against its pending map. Unlike the regex-intercept approach, text in the general channel can never accidentally match — approval requires the server to deliberately emit this specific event.

**两层隔离设计**：

- 普通文本走 `notifications/claude/channel` —— 模型看到、不影响权限；
- 权限回复走 `notifications/claude/channel/permission` —— **server 端 parse** "yes tbxkq" → 结构化 `{request_id, behavior}`；
- CC 不在本地 regex 匹配文本 —— **避免一句话"yes xxxxx"误触权限**。

server 必须**显式 opt-in**（声明 `claude/channel/permission` capability）才会变成权限通道。

## 9.3 wrapChannelMessage —— XML 注入防御

```ts
// channelNotification.ts:104
const SAFE_META_KEY = /^[a-zA-Z_][a-zA-Z0-9_]*$/

export function wrapChannelMessage(serverName, content, meta?): string {
  const attrs = Object.entries(meta ?? {})
    .filter(([k]) => SAFE_META_KEY.test(k))
    .map(([k, v]) => ` ${k}="${escapeXmlAttr(v)}"`)
    .join('')
  return `<${CHANNEL_TAG} source="${escapeXmlAttr(serverName)}"${attrs}>\n${content}\n</${CHANNEL_TAG}>`
}
```

注释（channelNotification.ts:97-104）：

> Meta keys become XML attribute NAMES — a crafted key like `x="" injected="y` would break out of the attribute structure. Only accept keys that look like plain identifiers. This is stricter than the XML spec (which allows `:`, `.`, `-`) but channel servers only send `chat_id`, `user`, `thread_ts`, `message_id` in practice.

3 重防御：

1. **key 白名单**：`/^[a-zA-Z_][a-zA-Z0-9_]*$/` —— 任何含 `"` 或空格的 key 直接丢；
2. **value escape**：`escapeXmlAttr(v)` —— `"` → `&quot;`；
3. **过滤而非 throw**：恶意 key 静默丢掉，**不让单个坏 key 阻断整个消息**。

[prompt-injection topic](../prompt-injection/) 讲过 XML 注入是 LLM 应用的常见攻击面——meta 直接进 system prompt 必须严格清洗。

## 9.4 6 层 gate

`gateChannelServer` (channelNotification.ts:191-316) 实现 6 层 gate：

```
1. capability    server 必须声明 experimental['claude/channel']
2. disabled      isChannelsEnabled() —— GrowthBook tengu_harbor 全局开关
3. auth          getClaudeAIOAuthTokens()?.accessToken —— OAuth only
4. policy        team/enterprise 必须 channelsEnabled: true
5. session       findChannelEntry —— --channels 命令行参数
6. marketplace + allowlist
   ├─ marketplace: pluginSource 必须匹配 --channels 里的 @marketplace
   └─ allowlist: {marketplace, plugin} 必须在 tengu_harbor_ledger
```

注释（channelNotification.ts:175-181）：

> Gate order: capability → runtime gate (tengu_harbor) → auth (OAuth only) → org policy → session --channels → allowlist.
> API key users are blocked at the auth layer — channels requires claude.ai auth; console orgs have no admin opt-in surface yet.

### 1. capability：`experimental['claude/channel']`

```ts
if (!capabilities?.experimental?.['claude/channel']) {
  return { action: 'skip', kind: 'capability', reason: '...' }
}
```

注释：

> Channel servers declare `experimental['claude/channel']: {}` (MCP's presence-signal idiom — same as `tools: {}`). Truthy covers `{}` and `true`; absent/undefined/explicit-`false` all fail.

`tools: {}` 是 MCP 的"我支持"信号——空对象也算 truthy。`false` 必须明确反对。

### 2. disabled：GrowthBook tengu_harbor killswitch

```ts
if (!isChannelsEnabled()) {
  return { action: 'skip', kind: 'disabled', ... }
}
```

```ts
// channelAllowlist.ts:51
export function isChannelsEnabled(): boolean {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_harbor', false)
}
```

**远程 killswitch**——发现问题可立即关掉所有用户的 channel 功能。注释：

> After capability so normal MCP servers never hit this path. Before auth/policy so the killswitch works regardless of session state.

### 3. auth：OAuth only

```ts
if (!getClaudeAIOAuthTokens()?.accessToken) {
  return { action: 'skip', kind: 'auth', reason: 'channels requires claude.ai authentication (run /login)' }
}
```

**API key 用户被挡** —— console 还没有 admin opt-in 界面，这是临时阻断。

### 4. policy：team/enterprise 必须显式 opt-in

```ts
const sub = getSubscriptionType()
const managed = sub === 'team' || sub === 'enterprise'
const policy = managed ? getSettingsForSource('policySettings') : undefined
if (managed && policy?.channelsEnabled !== true) {
  return { action: 'skip', kind: 'policy', ... }
}
```

注释：

> Default OFF — absent or false blocks. Keyed off subscription tier, not "policy settings exist" — a team org with zero configured policy keys (remote endpoint returns 404) is still a managed org and must not fall through to the unmanaged path.

**managed org 默认关闭**——必须 admin 显式 `channelsEnabled: true`。**判定按订阅类型**而不是"是否有 policy 配置"——零配置的 team org 也要走 managed 路径。

### 5. session：--channels 命令行声明

```ts
const entry = findChannelEntry(serverName, getAllowedChannels())
if (!entry) {
  return { action: 'skip', kind: 'session', reason: `server ${serverName} not in --channels list for this session` }
}
```

**每个 session 显式声明**——用户必须 `claude --channels plugin:telegram@anthropic` 启动才能让该 channel 注册。

注释：

> A server must be explicitly listed in --channels to push inbound this session — protects against a trusted server surprise-adding the capability.

防止**已 trust 的 server 升级后突然加了 channel capability**——还要二次确认。

### 6. marketplace 验证 + allowlist

`plugin:` kind entry 还要两步：

```ts
// 6a. marketplace 验证
const actual = pluginSource ? parsePluginIdentifier(pluginSource).marketplace : undefined
if (actual !== entry.marketplace) {
  return { action: 'skip', kind: 'marketplace', ... }
}

// 6b. allowlist
if (!entry.dev) {
  const { entries, source } = getEffectiveChannelAllowlist(sub, policy?.allowedChannelPlugins)
  if (!entries.some(e => e.plugin === entry.name && e.marketplace === entry.marketplace)) {
    return { action: 'skip', kind: 'allowlist', ... }
  }
}
```

注释（channelNotification.ts:260-275）：

> Marketplace verification: the tag is intent (plugin:slack@anthropic), the runtime name is just plugin:slack:X — could be slack@anthropic or slack@evil depending on what's installed. Verify they match before trusting the tag for the allowlist check below.

用户写 `--channels plugin:slack@anthropic` —— **tag 是意图**。但运行时 plugin 名只是 `plugin:slack:foo`，可能装的是 `slack@evil`。先验证才能信。

allowlist 来自 GrowthBook ledger `tengu_harbor_ledger` —— 平台维护一份"认证过的 channel plugin"清单。`team/enterprise` 可用 `allowedChannelPlugins` 覆盖。

`entry.dev` 是 `--dangerously-load-development-channels` —— 开发者本地 bypass，**但不影响 production 用户**。

### server-kind 永远 fail allowlist

```ts
} else {
  // server-kind: allowlist schema is {marketplace, plugin} — a server entry
  // can never match. Without this, --channels server:plugin:foo:bar would
  // match a plugin's runtime name and register with no allowlist check.
  if (!entry.dev) {
    return { action: 'skip', kind: 'allowlist', ... }
  }
}
```

allowlist schema 只支持 `{marketplace, plugin}` —— 裸 server name 永远不匹配。**只有 dev 模式可裸 server**——避免 `--channels foo` 绕过 allowlist。

## 9.5 permission 短 ID

### shortRequestId —— 5 letters

`channelPermissions.ts:140-152`：

```ts
export function shortRequestId(toolUseID: string): string {
  let candidate = hashToId(toolUseID)
  for (let salt = 0; salt < 10; salt++) {
    if (!ID_AVOID_SUBSTRINGS.some(bad => candidate.includes(bad))) {
      return candidate
    }
    candidate = hashToId(`${toolUseID}:${salt}`)
  }
  return candidate
}
```

设计目标：**手机能打、不出尴尬词**。

### 25 字母无 `l`

```ts
const ID_ALPHABET = 'abcdefghijkmnopqrstuvwxyz'  // 没有 'l'
```

注释（channelPermissions.ts:131-138）：

> 5 letters from a 25-char alphabet (a-z minus 'l' — looks like 1/I in many fonts). 25^5 ≈ 9.8M space, birthday collision at 50% needs ~3K simultaneous pending prompts, absurd for a single interactive session. Letters-only so phone users don't switch keyboard modes (hex alternates a-f/0-9 → mode toggles). Re-hashes with a salt suffix if the result contains a blocklisted substring — 5 random letters can spell things you don't want in a text message to your phone. toolUseIDs are `toolu_` + base64-ish; we hash rather than slice.

5 维设计：

1. **无 l** —— 字体里像 1 / I；
2. **25^5 ≈ 9.8M** —— 同时挂 3K 个 prompt 才 50% 碰撞——单 session 不可能；
3. **letters only** —— 手机不切数字键盘；
4. **5 字符** —— 短到能口播，长到不易输错；
5. **hash 而不是 slice** —— `toolUseID` 是 `toolu_<base64>`，slice 会带前缀。

### 脏词黑名单

```ts
// channelPermissions.ts:85-110
const ID_AVOID_SUBSTRINGS = [
  'fuck', 'shit', 'cunt', 'cock', 'dick', 'twat', 'piss',
  'crap', 'bitch', 'whore', 'ass', 'tit', 'cum', 'fag',
  'dyke', 'nig', 'kike', 'rape', 'nazi', 'damn', 'poo',
  'pee', 'wank', 'anus',
]
```

注释（channelPermissions.ts:81-83）：

> 5 random letters can spell things (Kenneth, in the launch thread: "this is why i bias to numbers, hard to have anything worse than 80085"). Non-exhaustive, covers the send-to-your-boss-by-accident tier. If a generated ID contains any of these, re-hash with a salt.

真实考量——**不能让用户给老板发"yes fucky"** 这种短信。Kenneth 在内部讨论里说"用数字最安全（80085 是最坏的）"——但坚持 letters，加黑名单兜底。

碰撞率：

> 7 length-3 × 3 positions × 25² + 15 length-4 × 2 × 25 + 2 length-5 ≈ 13,877 blocked IDs out of 9.8M — roughly 1 in 700 hits the blocklist. Cap at 10 retries; (1/700)^10 is negligible.

### FNV-1a hash

```ts
// channelPermissions.ts:112-128
function hashToId(input: string): string {
  let h = 0x811c9dc5
  for (let i = 0; i < input.length; i++) {
    h ^= input.charCodeAt(i)
    h = Math.imul(h, 0x01000193)
  }
  h = h >>> 0
  let s = ''
  for (let i = 0; i < 5; i++) {
    s += ID_ALPHABET[h % 25]
    h = Math.floor(h / 25)
  }
  return s
}
```

FNV-1a → uint32 → base-25 编码。不是密码学 hash —— 只要 **stable 短 ID** 就够。32 bits / log2(25) ≈ 6.9 字母——取 5 字母浪费一点熵但够。

### PERMISSION_REPLY_RE

```ts
export const PERMISSION_REPLY_RE = /^\s*(y|yes|n|no)\s+([a-km-z]{5})\s*$/i
```

注释（channelPermissions.ts:63-73）：

> 5 lowercase letters, no 'l' (looks like 1/I). Case-insensitive (phone autocorrect). No bare yes/no (conversational). No prefix/suffix chatter.
>
> CC generates the ID and sends the prompt. The SERVER parses the user's reply and emits notifications/claude/channel/permission with {request_id, behavior} — CC doesn't regex-match text anymore. Exported so plugins can import the exact regex rather than hand-copying it.

[a-km-z] 字符类——排除 l。**导出 regex 给 plugin 用**——避免每个 channel server 自己抄。

## 9.6 Kenneth 的"self-approve" 评估

`channelPermissions.ts:15-23` 长注释：

> Kenneth's "would this let Claude self-approve?": the approving party is the human via the channel, not Claude. But the trust boundary isn't the terminal — it's the allowlist (tengu_harbor_ledger). A compromised channel server CAN fabricate "yes <id>" without the human seeing the prompt. Accepted risk: a compromised channel already has unlimited conversation-injection turns (social-engineer over time, wait for acceptEdits, etc.); inject-then-self-approve is faster, not more capable. The dialog slows a compromised channel; it doesn't stop one.
> See PR discussion 2956440848.

**坦诚的威胁模型**：

- "Claude 能自己批准吗？" —— 不能，**人通过 channel 批**。
- 但 channel server 妥协怎么办？—— **可以伪造 yes**。
- 接受：妥协的 channel 已经有无限 injection 机会——快慢之别，不是能力之别。
- 真正信任边界：**allowlist（tengu_harbor_ledger）** —— 平台审核 plugin 的环节。

注释**直接 link 到 PR 讨论**让设计透明可追溯。

## 9.7 callbacks 工厂 + Map 闭包

`channelPermissions.ts:209-240`：

```ts
export function createChannelPermissionCallbacks(): ChannelPermissionCallbacks {
  const pending = new Map<string, (response: ChannelPermissionResponse) => void>()

  return {
    onResponse(requestId, handler) {
      const key = requestId.toLowerCase()
      pending.set(key, handler)
      return () => { pending.delete(key) }
    },

    resolve(requestId, behavior, fromServer) {
      const key = requestId.toLowerCase()
      const resolver = pending.get(key)
      if (!resolver) return false
      // Delete BEFORE calling — if resolver throws or re-enters, the
      // entry is already gone. Also handles duplicate events.
      pending.delete(key)
      resolver({ behavior, fromServer })
      return true
    },
  }
}
```

5 个细节：

### 1. Map 闭在工厂内

注释（channelPermissions.ts:196-201）：

> The pending Map is closed over — NOT module-level (per src/CLAUDE.md), NOT in AppState (functions-in-state causes issues with equality/serialization). Same lifetime pattern as `replBridgePermissionCallbacks`: constructed once per session inside a React hook, stable reference stored in AppState.

**3 个不放的地方**：
- module-level —— 多 session 共享 state 会污染；
- AppState 直接放 —— functions 序列化 / equality check 会爆；
- closure 内 —— 一个 React hook 创建，**stable reference 放 AppState**。

### 2. lowercase 双侧

```ts
const key = requestId.toLowerCase()
```

`onResponse` 和 `resolve` **都 lowercase**——避免 mixed-case 永不匹配。注释：

> shortRequestId always emits lowercase so this is a noop today, but the symmetry makes the contract explicit.

防御未来 caller。

### 3. delete BEFORE calling

```ts
pending.delete(key)
resolver({ behavior, fromServer })
```

注释：

> Delete BEFORE calling — if resolver throws or re-enters, the entry is already gone. Also handles duplicate events (second emission falls through — server bug or network dup, ignore).

防 resolver throw 时留下垃圾 entry，防 server 重发同 event 触发两次。

### 4. resolve 返回 boolean

`true` 表示**这是个我们追的 request**——caller 可以决定要不要 ack server。

### 5. unsubscribe 返回值

`onResponse` 返回 cleanup fn——caller 在 cancel 时调，避免 dead entry 堆积。

## 9.8 filterPermissionRelayClients —— 双 capability + allowlist

`channelPermissions.ts:177-194`：

```ts
export function filterPermissionRelayClients<T extends {...}>(
  clients: readonly T[],
  isInAllowlist: (name: string) => boolean,
): (T & { type: 'connected' })[] {
  return clients.filter((c): c is T & { type: 'connected' } =>
    c.type === 'connected' &&
    isInAllowlist(c.name) &&
    c.capabilities?.experimental?.['claude/channel'] !== undefined &&
    c.capabilities?.experimental?.['claude/channel/permission'] !== undefined,
  )
}
```

**3 个 ALL required**：
- connected（连上了）
- in allowlist（session opt-in 过）
- 双 capability（普通 + 权限）

注释（channelPermissions.ts:169-175）：

> The second capability is the server's explicit opt-in — a relay-only channel never becomes a permission surface by accident (Kenneth's "users may be unpleasantly surprised"). Centralized here so a future fourth condition lands once.

普通 channel server **不会意外变权限通道**——必须 server 显式声明第二个 capability。

中心化判定—— "future fourth condition lands once" 反映**演进意识**，不分散在多处。

## 9.9 elicitation —— 结构化用户表单

### 什么是 elicitation

MCP server 通过 JSON-RPC `request` 让 Claude Code **弹出表单 / URL**给用户：

```
Server                       Claude Code                       User
  |--- ElicitRequest ---->     |
  |  { message, schema }       |--- 显示表单 ----------------->
  |                            |                                |
  |                            |<--- {action: 'accept', content} ---|
  |<--- ElicitResult ----      |
  |  { action, content }       |
```

两种 mode：

| mode | 场景 | 例 |
|------|------|----|
| `form` | 结构化字段填写 | "请输入 API key" |
| `url` | 打开浏览器认证 | OAuth 第三方连接 |

### registerElicitationHandler

`elicitationHandler.ts:68`：

```ts
client.setRequestHandler(ElicitRequestSchema, async (request, extra) => {
  // 1. hook 优先（可程序化响应）
  const hookResponse = await runElicitationHooks(serverName, request.params, extra.signal)
  if (hookResponse) return hookResponse

  // 2. enqueue 到 AppState
  const response = new Promise<ElicitResult>(resolve => {
    setAppState(prev => ({
      ...prev,
      elicitation: {
        queue: [...prev.elicitation.queue, {
          serverName,
          requestId: extra.requestId,
          params: request.params,
          signal: extra.signal,
          waitingState,
          respond: (result) => resolve(result),
        }],
      },
    }))
    extra.signal.addEventListener('abort', onAbort, { once: true })
  })

  const rawResult = await response
  // 3. result hook（可改 result）
  const result = await runElicitationResultHooks(serverName, rawResult, extra.signal, mode, elicitationId)
  return result
})
```

3 步：

### 1. Pre-elicitation hook

`runElicitationHooks` 让用户定义的 hook **程序化响应**——不弹 UI 直接答。

例：CI 环境跑 hook 自动 `accept` 默认值——non-interactive 也能完成测试。

### 2. AppState queue + Promise

请求进 `AppState.elicitation.queue` ——React UI 渲染 dialog。用户操作触发 `respond(result)` resolve Promise。

`abortSignal` 取消时返 `{action: 'cancel'}`。

### 3. Post-elicitation hook

`runElicitationResultHooks` —— hook 看到用户答完可以**改答案**或 block（返 `decline`）。

### url mode + completion notification

URL 模式特殊：用户先打开浏览器（如 OAuth 同意）—— Claude Code **不知道**他什么时候完成。

server 通过 `ElicitationCompleteNotification` 主动通知：

```ts
client.setNotificationHandler(ElicitationCompleteNotificationSchema, (notification) => {
  const { elicitationId } = notification.params
  setAppState(prev => {
    const idx = findElicitationInQueue(prev.elicitation.queue, serverName, elicitationId)
    if (idx === -1) return prev
    const queue = [...prev.elicitation.queue]
    queue[idx] = { ...queue[idx]!, completed: true }
    return { ...prev, elicitation: { queue } }
  })
})
```

UI 看到 `completed: true` —— 把"等待用户在浏览器操作"的 spinner 换成"已完成、点击继续"按钮。

注释（elicitationHandler.ts:173-174）：

> Sets `completed: true` on the matching queue event; the dialog reacts to this flag.

### waitingState —— 两阶段 dialog

```ts
const waitingState: ElicitationWaitingState | undefined =
  elicitationId ? { actionLabel: 'Skip confirmation' } : undefined
```

url mode + elicitationId 时——dialog 分两阶段：

1. **第一阶段**：显示"点这里打开 URL" → 用户点击；
2. **第二阶段**：显示"等待中... [Skip confirmation]" → 用户可强制 skip。

`onWaitingDismiss` callback —— UI dismiss waiting state 时调。

### 错误重试 elicitation（-32042）

某些 MCP server 在工具调用失败时返 `-32042` error code + elicitation params —— **让用户决定要不要重试**。

这种场景 `'accept'` action 是 no-op —— 重试由 `onWaitingDismiss('retry')` 驱动。

### setRequestHandler try/catch

```ts
try {
  client.setRequestHandler(ElicitRequestSchema, async (...) => { ... })
} catch {
  // Client wasn't created with elicitation capability - nothing to register
  return
}
```

**没有 elicitation capability 的 client 调 setRequestHandler 会 throw** —— catch 静默忽略。capability negotiation 后期决定的，client 创建时还不知道是否需要 handler。

## 9.10 hook 系统

### executeElicitationHooks

`elicitationHandler.ts:214-257`：

```ts
export async function runElicitationHooks(serverName, params, signal): Promise<ElicitResult | undefined> {
  const { elicitationResponse, blockingError } = await executeElicitationHooks({
    serverName,
    message: params.message,
    requestedSchema: 'requestedSchema' in params ? params.requestedSchema : undefined,
    signal,
    mode,
    url,
    elicitationId,
  })

  if (blockingError) return { action: 'decline' }
  if (elicitationResponse) return { action, content: elicitationResponse.content }
  return undefined
}
```

3 种 hook 返回：

- **`blockingError`** —— hook 报错并阻断 → action: 'decline'；
- **`elicitationResponse`** —— hook 提供答案 → 直接返；
- **`undefined`** —— hook 不响应 → 继续走 UI 流程。

### executeElicitationResultHooks

用户答完后调——hook 可以**审计 / 改 / 拒**：

```ts
const { elicitationResultResponse, blockingError } = await executeElicitationResultHooks({
  serverName, action, content, signal, mode, elicitationId,
})

if (blockingError) {
  notify('decline')
  return { action: 'decline' }
}

const finalResult = elicitationResultResponse
  ? { action: ..., content: ... ?? result.content }
  : result

notify(finalResult.action)
return finalResult
```

注释（elicitationHandler.ts:259-263）：

> Run ElicitationResult hooks after the user has responded, then fire a `elicitation_response` notification. Returns a (potentially modified) ElicitResult — hooks may override the action/content or block the response.

### notification 即使错误也发

```ts
} catch (error) {
  logMCPError(serverName, `ElicitationResult hook error: ${error}`)
  // Fire notification even on error
  void executeNotificationHooks({ message: '...', notificationType: 'elicitation_response' })
  return result
}
```

hook 报错不阻断 elicitation 流程——但**通知 hook 必发**保证 observability。

## 9.11 几个隐性设计判断

### 1. channel 是"标准 MCP + 一个 notification method"

不是新协议——MCP 的 `notifications/*` 扩展点正好用上。

### 2. 普通文本与权限回复分两个 method

避免一句 "yes xxxxx" 误触权限—— **server 必须显式 emit 结构化 event**。

### 3. wrapChannelMessage 三重防注入

key 白名单 + value escape + 静默过滤——XML 注入完全堵死。

### 4. 6 层 gate

capability → killswitch → auth → policy → session → marketplace+allowlist —— **每层独立**，问题随时可关任意层。

### 5. session opt-in 防 capability 突袭

trusted server 升级偷加 channel cap → 不会自动生效，必须用户 `--channels` 显式声明。

### 6. marketplace 验证防替身攻击

`--channels plugin:slack@anthropic` 是意图——实际装的是 `slack@evil` 时拒绝注册。

### 7. server-kind 永远 fail allowlist

allowlist schema 只支持 plugin —— 裸 server name 必须 `--dangerously` 才注册。

### 8. 5 letter ID 多重考量

无 l + 25^5 空间 + letters-only + 5 长 + hash 不 slice —— 手机能打 + 不碰撞 + 不出错。

### 9. 脏词黑名单 hard-coded

"yes fucky" 给老板的 worst case —— 13877/9.8M ≈ 1/700 命中重 hash 10 次几乎不会失败。

### 10. PERMISSION_REPLY_RE 导出给 plugin

避免每个 channel server 重抄 regex —— 单一来源。

### 11. Kenneth 威胁模型公开

接受 "channel server 妥协可伪造 yes" —— 公开记录在源码注释，决策可追溯。

### 12. callbacks Map 闭包内

不放 module / 不放 AppState —— React hook 创建一次，stable reference 进 AppState。

### 13. delete BEFORE calling

resolver throw / 重发都不留垃圾。

### 14. 双 capability filter

普通 channel 不会意外变权限通道——server 必须显式 opt-in。

### 15. elicitation hook 在 UI 前后

pre-hook 程序化答（CI 场景）+ post-hook 审计/改/拒 —— 灵活注入点。

### 16. url mode 两阶段 dialog

waitingState 让用户能 skip——浏览器认证不可控时的 escape hatch。

### 17. elicitation completion 主动通知

URL 模式 server 主动 push 完成—— UI 切状态而不是 poll。

### 18. setRequestHandler try/catch silent

client 没有 elicitation cap 时静默不注册——避免 capability negotiation 时序问题。

### 19. hook error 不阻断但必发 notification

elicitation 流程不能因 hook 报错卡住——observability 单独保证。

## 9.12 与其它专题串联

- **[swarm 08 SendUserMessage](../swarm/08-SendUserMessage与SleepTool.md)**：channel 入消息走 SleepTool 1s wake 机制；
- **[swarm 11 hooks](../swarm/11-hooks机制详解.md)**：elicitation/elicitationResult hook 注入点；
- **[prompt-injection 03 system tags](../prompt-injection/)**：`<channel>` 标签设计 + XML attr escape；
- **[bashtool 04 permission](../bashtool/04-permission决策核心.md)**：channel permission relay 是 BridgePermissionCallbacks 的姐妹模式。

## 9.13 小结

- channel = MCP server + `notifications/claude/channel` 推消息能力；
- 两个 inbound：普通消息（`<channel>` 包） vs 结构化权限回复（避免文本误触）；
- `wrapChannelMessage` 三重防 XML 注入：key 白名单 + value escape + 静默过滤；
- 6 层 gate：capability → killswitch → auth → policy → session → marketplace+allowlist；
- session `--channels` 防止 server 升级偷加 capability；
- marketplace 验证防 `slack@evil` 替身；
- server-kind 必须 `--dangerously` —— allowlist schema 只支持 plugin；
- 5-letter ID 设计：无 l + 25^5 空间 + letters-only + hash 不 slice + 脏词黑名单；
- `PERMISSION_REPLY_RE` 导出给 plugin 避免重抄；
- Kenneth "self-approve" 威胁模型公开记录——接受妥协 channel 风险；
- callbacks Map 闭包内（不 module / 不 AppState），delete BEFORE calling 防 throw；
- filterPermissionRelayClients 三重过滤（connected + allowlist + 双 capability）；
- elicitation 是 MCP 结构化表单/URL 请求；
- hook 系统：pre-hook 程序化答 + post-hook 审计/改/拒；
- URL mode 两阶段 dialog + 主动 completion 通知；
- setRequestHandler 静默 try/catch 处理 capability 缺失；
- hook error 不阻断流程但保证 notification 必发。

下一篇 → [10 连接管理与生命周期](./10-连接管理与生命周期.md)
