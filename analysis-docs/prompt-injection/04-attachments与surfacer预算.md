# 04 attachments 与 surfacer 预算

> Claude Code 把"友方注入"（系统主动塞进上下文的内容）也当成需要严格约束的资源。本篇拆解为什么、用什么机制、配额是多少。

## 4.1 友方注入 vs 敌方注入：为什么用同一套机制处理

到目前为止讲的 prompt-injection 防御都是**反"敌方注入"**：攻击者通过文件 / Web / MCP 等通道把恶意内容塞进上下文。

但 Claude Code 自己也在**主动塞内容**：

- 每隔几个 turn 提醒一次 todo 列表状态；
- plan mode 下定期注入阶段说明；
- 自动检索"相关记忆"并把它们附加到下轮上下文；
- 启动时塞一份 `ls` 当前目录的结果；
- 进入子代理时塞 agent memory；
- 每次发请求前注入 `<env>` / `<git-status>` / current date / model info。

这些是 **"友方注入"**——系统出于"让模型工作更好"的目的主动放进上下文的内容。它们和敌方注入有几个共同特征：

| 特征 | 友方注入 | 敌方注入 |
|---|---|---|
| 不是模型自己生成的 | ✓ | ✓ |
| 不是用户主动发的 | ✓ | ✓ |
| 通过"工具结果 / attachment / user message"通道进入 | ✓ | ✓ |
| 试图引导模型行为 | ✓（引导往任务方向） | ✓（引导往攻击方向） |
| 消耗 context window 预算 | ✓ | ✓ |

所以从治理视角看，它们应该用同一套机制管理——这就是为什么 `<system-reminder>` 既给 plan 提醒用、也给 FileRead 的 malware 警告用、也给 skill discovery 用。

但友方注入有一个**关键不同**：

> **它的频率和大小是可控的**——是 Claude Code 自己决定要不要注入、注入多少、注入多频繁。

于是设计上有了：**给友方注入加配额**。这一篇拆解配额机制。

## 4.2 配额的三个维度

`src/utils/attachments.ts:254-293`：

```ts
export const TODO_REMINDER_CONFIG = {
  TURNS_SINCE_WRITE: 10,
  TURNS_BETWEEN_REMINDERS: 10,
} as const

export const PLAN_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,
} as const

export const AUTO_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,
} as const

const MAX_MEMORY_LINES = 200
const MAX_MEMORY_BYTES = 4096

export const RELEVANT_MEMORIES_CONFIG = {
  MAX_SESSION_BYTES: 60 * 1024,
} as const

export const VERIFY_PLAN_REMINDER_CONFIG = {
  TURNS_BETWEEN_REMINDERS: 10,
} as const
```

三类配额：

| 维度 | 控制什么 | 典型值 |
|---|---|---|
| **频率**（TURNS_BETWEEN_*） | 每隔多少 turn 才能再注入一次 | 5-10 turns |
| **单次大小**（MAX_MEMORY_BYTES / MAX_MEMORY_LINES） | 单次注入的字节/行数上限 | 4 KB / 200 行 |
| **会话累积**（MAX_SESSION_BYTES） | 整个会话的累计注入字节上限 | 60 KB |

下面分别看每一类。

## 4.3 频率配额：避免"骚扰"

`TURNS_BETWEEN_*` 系列控制提醒类注入的最小间隔。例如：

- `TODO_REMINDER_CONFIG.TURNS_BETWEEN_REMINDERS = 10`：todo 提醒最少隔 10 turn 才能再注入
- `PLAN_MODE_ATTACHMENT_CONFIG.TURNS_BETWEEN_ATTACHMENTS = 5`：plan mode 提醒最少隔 5 turn
- `VERIFY_PLAN_REMINDER_CONFIG.TURNS_BETWEEN_REMINDERS = 10`：verify plan 提醒最少隔 10 turn

为什么要限频？两个原因：

### 原因 1：避免 prompt 噪声压过用户原话

如果每个 turn 都注入 5 条系统提醒、外加 todo 状态、外加相关记忆，那么用户原话在 context 里的占比会被稀释。模型可能更关注"系统反复强调"的内容而忽略真实任务。

### 原因 2：避免训练负样本

模型在训练时见过"被反复提醒同一件事"的数据，会形成对该 pattern 的特殊响应——可能是 ignore（学到"提醒可以忽略"），也可能是 over-attend（学到"提醒就该立即响应"）。两者都会影响主任务。限频是为了让提醒**保持稀缺，从而保持有效**。

### 应用举例（`attachments.ts:1204` 附近）

```ts
turnCount < PLAN_MODE_ATTACHMENT_CONFIG.TURNS_BETWEEN_ATTACHMENTS
```

注入函数会先检查这个条件，达不到就跳过。这是个简单的 throttle。

### 全提醒 vs 稀疏提醒（FULL_REMINDER_EVERY_N）

`FULL_REMINDER_EVERY_N_ATTACHMENTS = 5` 表示：每 5 次注入里 4 次用"稀疏版"，第 5 次用"完整版"。

```ts
// attachments.ts:1227 附近
PLAN_MODE_ATTACHMENT_CONFIG.FULL_REMINDER_EVERY_N_ATTACHMENTS === ...
```

稀疏版只包含核心指令、完整版包含全部上下文。这是另一个**节流维度**：即使到了"该注入"的 turn，也不是每次都注入"完整版"——大多数时候用精简版省 token。

## 4.4 单次大小配额：MAX_MEMORY_BYTES = 4096

`attachments.ts:269-277`：

```ts
const MAX_MEMORY_LINES = 200
// Line cap alone doesn't bound size (200 × 500-char lines = 100KB).  The
// surfacer injects up to 5 files per turn via <system-reminder>, bypassing
// the per-message tool-result budget, so a tight per-file byte cap keeps
// aggregate injection bounded (5 × 4KB = 20KB/turn).  Enforced via
// readFileInRange's truncateOnByteLimit option.  Truncation means the
// most-relevant memory still surfaces: the frontmatter + opening context
// is usually what matters.
const MAX_MEMORY_BYTES = 4096
```

注释把推理过程写得非常清楚，值得逐条拆：

### 推理 1：line cap 不够

```
"Line cap alone doesn't bound size (200 × 500-char lines = 100KB)."
```

最初可能只想限 200 行。但一行可以非常长——某些 minified 代码、JSON、protobuf 文本——200 × 500 字符 = 100KB，一次注入塞 100KB 进上下文就过了。

### 推理 2：bypass per-message tool-result budget

```
"The surfacer injects up to 5 files per turn via <system-reminder>,
 bypassing the per-message tool-result budget"
```

正常 tool_result 有自己的字节预算（FileRead 默认 ~25K tokens cap）。但 surfacer 走的是 `<system-reminder>` 通道，**绕过**了 tool_result 的预算。这是个设计上的不对称：

- 用户主动调 FileRead → 受 tool_result 预算限制；
- 系统自动塞相关记忆 → 不受 tool_result 预算限制，可以塞 5 个文件。

为了不让这个不对称被滥用，**单独**给 surfacer 加了 4KB/文件 的上限。

### 推理 3：4KB × 5 = 20KB / turn

```
"a tight per-file byte cap keeps aggregate injection bounded (5 × 4KB = 20KB/turn)"
```

5 个文件每个 4KB → 整个 turn 最多注入 20KB。这是单 turn 注入的硬上限。

### 推理 4：截断而不是丢弃

```
"Truncation means the most-relevant memory still surfaces:
 the frontmatter + opening context is usually what matters."
```

文件超过 4KB 时**截断**而不是**跳过**。理由：

- 既然 surfacer 已经把它选为"相关"，至少要让模型看到开头；
- 大多数 markdown 笔记的关键信息在前面（frontmatter、第一段、第一个 section）；
- 截断后会附加一句 "this memory file was truncated, use Read tool to view"（见 `attachments.ts:2306`），告诉模型如果需要可以主动 FileRead。

## 4.5 会话累积配额：MAX_SESSION_BYTES = 60 KB

`attachments.ts:279-289`：

```ts
export const RELEVANT_MEMORIES_CONFIG = {
  // Per-turn cap (5 × 4KB = 20KB) bounds a single injection, but over a
  // long session the selector keeps surfacing distinct files — ~26K tokens/
  // session observed in prod.  Cap the cumulative bytes: once hit, stop
  // prefetching entirely.  Budget is ~3 full injections; after that the
  // most-relevant memories are already in context.  Scanning messages
  // (rather than tracking in toolUseContext) means compact naturally
  // resets the counter — old attachments are gone from context, so
  // re-surfacing is valid.
  MAX_SESSION_BYTES: 60 * 1024,
} as const
```

继续逐条解释：

### 为什么单 turn 配额还不够

```
"Per-turn cap bounds a single injection, but over a long session
 the selector keeps surfacing distinct files"
```

一次 20KB 不大。但如果会话很长——每隔几个 turn 都注入新一批相关记忆——累积起来就大了。生产观察到 **~26K tokens/session**（约 100KB）的相关记忆注入。

### 60KB 上限怎么定

```
"Budget is ~3 full injections; after that the most-relevant memories
 are already in context."
```

60KB ≈ 3 次满载注入（3 × 20KB）。设计假设：

- 3 次以内，selector 还在发现新的"相关记忆"；
- 3 次以上，最相关的几份基本都已经塞进上下文了，新塞的边际效益骤降。

所以到了 60KB 直接**停止 prefetch**——不再后台异步检索新记忆，本 session 余下时间不会再有新的 surfacer 注入。

### 上限的检查点

`attachments.ts:2382-2385`：

```ts
const surfaced = collectSurfacedMemories(messages)
if (surfaced.totalBytes >= RELEVANT_MEMORIES_CONFIG.MAX_SESSION_BYTES) {
  return undefined
}
```

`startRelevantMemoryPrefetch` 在启动新 prefetch 之前先扫描消息历史，统计已注入的字节数。够了就返回 undefined（"不启动"）。

### 巧妙：扫消息而不是 ctx counter

```
"Scanning messages (rather than tracking in toolUseContext) means
 compact naturally resets the counter — old attachments are gone
 from the compacted transcript, so re-surfacing is valid again."
```

这里有个**设计巧思**值得记住：

- ❌ 朴素做法：在 `toolUseContext` 里维护一个 `surfacedBytesCounter`，每次注入累加；
- ✅ 实际做法：每次需要决策时，**重新扫描** message 历史，统计还在 context 里的 surfacer attachment 总字节。

为什么后者更好？因为 Claude Code 有 **context 压缩**机制（compact）。压缩会把老消息总结成短文本——老的 surfacer attachment 在压缩后就消失了。

- 朴素做法：counter 仍是历史累加值，会错以为还在配额耗尽；
- 扫消息做法：自然就把"被压缩掉的注入"排除了，等于 **compact 重置 counter**。

这是一个 "**让数据结构反映真实状态**" 的优雅设计——不需要维护单独的 counter，去掉了 counter 失同步的可能。

### `collectSurfacedMemories` 的实现

`attachments.ts:2251-2266`：

```ts
export function collectSurfacedMemories(messages: ReadonlyArray<Message>): {
  paths: Set<string>
  totalBytes: number
} {
  const paths = new Set<string>()
  let totalBytes = 0
  for (const m of messages) {
    if (m.type === 'attachment' && m.attachment.type === 'relevant_memories') {
      for (const mem of m.attachment.memories) {
        paths.add(mem.path)
        totalBytes += mem.content.length
      }
    }
  }
  return { paths, totalBytes }
}
```

简单的线性扫描。返回两个东西：
- `paths`：已注入过的文件路径（用于 selector 去重，避免重复选）
- `totalBytes`：累计字节（用于配额检查）

注意 `paths` 也是这一招的好处——同样的设计无 counter 维护，跟随消息历史天然演化。

## 4.6 配额怎么应用到具体注入

把上面的几类配额组合起来看，一次"相关记忆"注入要过多重检查：

```
                       用户输入 → query loop 准备下一轮 API 调用
                                          │
                                          ▼
                  ┌──────────────────────────────────────────┐
                  │ startRelevantMemoryPrefetch(messages, ctx)│
                  └──────────────────────┬───────────────────┘
                                         │
                          ┌──────────────┴──────────────┐
                          │  Check 1: 功能开关          │
                          │  isAutoMemoryEnabled() &&    │
                          │  tengu_moth_copse feature   │
                          └──────────────┬──────────────┘
                                         │ pass
                                         ▼
                          ┌─────────────────────────────┐
                          │  Check 2: 用户输入足够长     │
                          │  /\s/.test(input.trim())    │
                          │  (单词输入不触发检索)         │
                          └──────────────┬──────────────┘
                                         │ pass
                                         ▼
                          ┌─────────────────────────────┐
                          │  Check 3: 会话累积配额       │
                          │  surfaced.totalBytes        │
                          │   < MAX_SESSION_BYTES (60KB)│
                          └──────────────┬──────────────┘
                                         │ pass
                                         ▼
                          ┌─────────────────────────────┐
                          │  启动后台 prefetch promise   │
                          │  (异步，不阻塞主请求)         │
                          └──────────────┬──────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │  findRelevantMemories       │
                          │  (语义检索 + ranker)         │
                          └──────────────┬──────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │  filter:                    │
                          │   !readFileState.has(path)  │
                          │   !alreadySurfaced.has(path)│
                          │  .slice(0, 5)               │ ← 单 turn 5 文件上限
                          └──────────────┬──────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │  readMemoriesForSurfacing   │
                          │  per-file:                  │
                          │   MAX_MEMORY_LINES=200      │ ← 单文件行数上限
                          │   MAX_MEMORY_BYTES=4096     │ ← 单文件字节上限
                          │   truncateOnByteLimit=true  │
                          └──────────────┬──────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │  返回 relevant_memories      │
                          │   attachment                │
                          └──────────────┬──────────────┘
                                         │
                                         ▼
                          ┌─────────────────────────────┐
                          │  normalize → ensureSR wrap  │
                          │  → smoosh 到 tool_result    │
                          └─────────────────────────────┘
                          (进入 [01-system-reminder] 管线)
```

可以看到，一次注入要过 **3 层硬阀** + **2 层软节流**：

- 硬阀：feature gate、session 配额、单 turn 5-file 上限
- 软节流：单文件字节/行数截断（不丢但截）

## 4.7 友方注入与敌方注入治理的差异点

虽然用同一套 `<system-reminder>` 标签机制，但两类注入在**配额管理**上完全不同：

| 维度 | 友方注入 | 敌方注入（FileRead 等） |
|---|---|---|
| 单次大小限制 | 严格（4KB/文件） | 工具自己的预算（FileRead ~25K tokens） |
| 单 turn 总量限制 | 严格（20KB） | 受 context window 间接限制 |
| 会话总量限制 | 严格（60KB） | 无 |
| 频率限制 | 严格（5-10 turn 间隔） | 无（用户/模型想读就读） |
| 截断策略 | 截断后注明 | 截断（FileRead）或拒绝（超 token cap） |
| 重置策略 | compact 自动重置 | 无重置 |

理由：

- **友方注入**自己决定，所以可以严格自控；
- **敌方/外部注入**是任务驱动的（模型/用户主动调 FileRead），不能武断限频，否则任务完不成。

但**敌方注入有另外两道护栏**（[02 FileRead 双护栏](./02-FileRead双护栏.md)：行号前缀 + malware reminder）——配额松、护栏紧。

## 4.8 一句话总结

- Claude Code 把"友方注入"也当成需要严格治理的资源；
- 三个维度的配额：**频率** (5-10 turn)、**单次大小** (4KB/file)、**会话累积** (60KB)；
- 配额计算用扫描消息历史而非 counter，**让 compact 自然重置**——这是漂亮的"数据结构反映真实状态"设计；
- 与敌方注入用同一标签机制（`<system-reminder>`），但**配额完全独立**：友方紧、敌方松，因为后者有护栏（行号前缀+malware reminder）兜底。

下一篇 → [05 外部工具内容隔离](./05-外部工具内容隔离.md)
