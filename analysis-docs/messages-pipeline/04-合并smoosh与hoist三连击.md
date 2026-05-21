# 04 合并 smoosh 与 hoist 三连击

> normalize 主循环里的合并不是一个函数——是 6 层嵌套的函数族。每一层做一件具体的事，且大多对应一个 A/B 实验。本篇把这套"积木"拆到底，让"为什么需要这么多层"变清楚。

## 4.1 为什么需要合并

API 协议层有几个硬约束：

1. **连续 user message 不允许**（至少 Bedrock 是这样；1P API 会合并）。两条相邻 user 必须合并成一条；
2. **tool_result 必须在 content[] 最前面**（"tool result must follow tool use" 否则 400）；
3. **tool_use ↔ tool_result 必须配对**——assistant 里的每个 tool_use 都得在下一条 user 里有对应 tool_result；
4. **相邻 text block 会被服务器拼接**（中间不加分隔符）——`"2+2"` + `"3+3"` 会变成 `"2+23+3"`。

加上几个软约束（不是协议要求，但模型行为受影响）：

5. **`<system-reminder>` 文本不能作为 tool_result 的 sibling**——会泄露 `Human:` 模式到训练数据；
6. **tool_reference content block 不能和 text/image 混在 tool_result 内**——server ValueError。

这些约束加起来意味着：normalize 必须把"逻辑上多条独立消息"**重塑**成"协议上合法的 message"。这就是合并族函数干的事。

## 4.2 合并族函数地图

按调用关系画一张图：

```
                normalize 主循环 (forEach + switch)
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
       user case     attachment    后处理段 pass 6
            │         case            │
            │             │             │
            ▼             ▼             ▼
   mergeUserMessages       mergeAdjacentUserMessages
            │                     │
            │   (内部走一遍 messages 数组)
            │             │
            └──────┬──────┘
                   │ calls
                   ▼
        ┌────────────────────────┐
        │ mergeUserContentBlocks │  ← 真正"合并 content[] 数组"的入口
        └───────────┬────────────┘
                    │
        ┌───────────┴───────────┬─────────────────────┐
        │                       │                     │
        ▼                       ▼                     ▼
joinTextAtSeam            smooshIntoToolResult   hoistToolResults
(在 mergeUserMessages 里  (在 mergeUserContentBlocks (在 mergeUserMessages
 直接调，预处理 a content)  里有条件调)              里合并完后调，最终重排)
```

外加一个**特殊版本** `mergeUserMessagesAndToolResults`：attachment case 用，因为 attachment 自己可能带 tool_result 引用（如 `async_hook_response`）。

## 4.3 顶层：`mergeAdjacentUserMessages`

`messages.ts:2451-2464`：

```ts
function mergeAdjacentUserMessages(
  msgs: (UserMessage | AssistantMessage)[],
): (UserMessage | AssistantMessage)[] {
  const out: (UserMessage | AssistantMessage)[] = []
  for (const m of msgs) {
    const prev = out.at(-1)
    if (m.type === 'user' && prev?.type === 'user') {
      out[out.length - 1] = mergeUserMessages(prev, m) // lvalue — can't use .at()
    } else {
      out.push(m)
    }
  }
  return out
}
```

朴素的单遍扫描：遇到"上一条也是 user"就合并。这是后处理段 pass 6 的核心——前面 forEach 里已经做了"merge to prev"的合并，这里是**全量再走一遍**确保**没有任何遗漏**。

为什么需要两道？因为后处理段会**先做 filter**（filterOrphanedThinking / filterWhitespace 等）——filter 可能让两条原本中间隔着 assistant 的 user 变得相邻。比如：

```
[user, assistant(只含 thinking), user]
   ↓ filterOrphanedThinking
[user, user]
   ↓ mergeAdjacent
[merged user]
```

如果只在 forEach 里合并，filter 后产生的相邻 user 就会漏掉。

## 4.4 单对合并：`mergeUserMessages`

`messages.ts:2411-2449`：

```ts
export function mergeUserMessages(a: UserMessage, b: UserMessage): UserMessage {
  const lastContent = normalizeUserTextContent(a.message.content)
  const currentContent = normalizeUserTextContent(b.message.content)
  if (feature('HISTORY_SNIP')) {
    const { isSnipRuntimeEnabled } = require('../services/compact/snipCompact.js')
    if (isSnipRuntimeEnabled()) {
      return {
        ...a,
        isMeta: a.isMeta && b.isMeta ? (true as const) : undefined,
        uuid: a.isMeta ? b.uuid : a.uuid,
        message: {
          ...a.message,
          content: hoistToolResults(
            joinTextAtSeam(lastContent, currentContent),
          ),
        },
      }
    }
  }
  return {
    ...a,
    uuid: a.isMeta ? b.uuid : a.uuid,
    message: {
      ...a.message,
      content: hoistToolResults(joinTextAtSeam(lastContent, currentContent)),
    },
  }
}
```

合并两条 user message。处理 4 件事：

### 4.4.1 内容字符串/数组规范化

```ts
const lastContent = normalizeUserTextContent(a.message.content)
const currentContent = normalizeUserTextContent(b.message.content)
```

`normalizeUserTextContent`（`messages.ts:2485-2492`）：

```ts
function normalizeUserTextContent(
  a: string | ContentBlockParam[],
): ContentBlockParam[] {
  if (typeof a === 'string') {
    return [{ type: 'text', text: a }]
  }
  return a
}
```

把字符串 content 规范成单元素数组——这样后续合并不用区分 string/array 两种 case。

### 4.4.2 接缝处插入 `\n` （joinTextAtSeam）

`messages.ts:2505-2515`：

```ts
function joinTextAtSeam(
  a: ContentBlockParam[],
  b: ContentBlockParam[],
): ContentBlockParam[] {
  const lastA = a.at(-1)
  const firstB = b[0]
  if (lastA?.type === 'text' && firstB?.type === 'text') {
    return [...a.slice(0, -1), { ...lastA, text: lastA.text + '\n' }, ...b]
  }
  return [...a, ...b]
}
```

注释（`messages.ts:2494-2503`）：

> The API concatenates adjacent text blocks in a user message without a separator, so two queued prompts `"2 + 2"` + `"3 + 3"` would otherwise reach the model as `"2 + 23 + 3"`.
>
> Blocks stay separate; the `\n` goes on a's side so no block's `startsWith` changes — `smooshSystemReminderSiblings` classifies via `startsWith('<system-reminder>')`, and prepending to b would break that when b is an SR-wrapped attachment.

两个关键点：

1. **API 把相邻 text block 无分隔符拼接**——这是一个隐性的协议行为。如果不在接缝处加 `\n`，模型看到的就是 `"2+23+3"` 这种乱码；
2. **`\n` 加在 a 的末尾、不在 b 的开头**——因为下游 `smooshSystemReminderSiblings` 用 `startsWith('<system-reminder>')` 识别 SR-text。如果 `\n` 加到 b 的开头，原本 `<system-reminder>...` 开头的 block 就变成 `\n<system-reminder>...`，startsWith 失效。

这是个**极其细的设计**——一个换行符的位置都要为下游分类器的判定 contract 让步。这种细节在工业代码里值得记下。

### 4.4.3 调 `hoistToolResults`

`messages.ts:2470-2483`：

```ts
function hoistToolResults(content: ContentBlockParam[]): ContentBlockParam[] {
  const toolResults: ContentBlockParam[] = []
  const otherBlocks: ContentBlockParam[] = []

  for (const block of content) {
    if (block.type === 'tool_result') {
      toolResults.push(block)
    } else {
      otherBlocks.push(block)
    }
  }

  return [...toolResults, ...otherBlocks]
}
```

注释（`messages.ts:2467-2469`）：

> In the content[] list on a UserMessage, tool_result blocks **must come first** to avoid "tool result must follow tool use" API errors.

把所有 `tool_result` block 移到前面，其它（text / image / 等）放后面。原顺序在两个分组内保持。

### 4.4.4 uuid 保留策略

```ts
uuid: a.isMeta ? b.uuid : a.uuid,
```

注释（`messages.ts:2441-2443`）：

> Preserve the non-meta message's uuid so [id:] tags (derived from uuid) stay stable across API calls (meta messages like system context get fresh uuids each call)

合并时**保留非 meta 的那一条的 uuid**。因为 [id:xxx] tag 是从 uuid 派生的——如果 meta message（每轮重新构造）的 uuid 覆盖了真用户 message 的 uuid，那 tag 在多轮调用间就漂移了。

### 4.4.5 isMeta 合并规则（HISTORY_SNIP 路径）

```ts
isMeta: a.isMeta && b.isMeta ? (true as const) : undefined,
```

注释（`messages.ts:2414-2421`）：

> A merged message is only meta if ALL merged messages are meta. If any operand is real user content, the result must not be flagged isMeta (so [id:] tags get injected and it's treated as user-visible content).

"如果合并的所有消息都是 meta，结果才是 meta；只要有一个是真用户内容，结果就不是 meta"。这是合理的：meta 状态是"系统注入的标记"，混入真用户内容后整体就不再是纯系统注入。

## 4.5 内容数组合并：`mergeUserContentBlocks`

`messages.ts:2600-2647`：

```ts
export function mergeUserContentBlocks(
  a: ContentBlockParam[],
  b: ContentBlockParam[],
): ContentBlockParam[] {
  const lastBlock = last(a)
  if (lastBlock?.type !== 'tool_result') {
    return [...a, ...b]
  }

  if (!checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_chair_sermon')) {
    // Legacy (ungated) smoosh: only string-content tool_result + all-text
    // siblings → joined string.
    if (
      typeof lastBlock.content === 'string' &&
      b.every(x => x.type === 'text')
    ) {
      const copy = a.slice()
      copy[copy.length - 1] = smooshIntoToolResult(lastBlock, b)!
      return copy
    }
    return [...a, ...b]
  }

  // Universal smoosh (gated): fold all non-tool_result block types (text,
  // image, document, search_result) into tool_result.content. tool_result
  // blocks stay as siblings (hoisted later by hoistToolResults).
  const toSmoosh = b.filter(x => x.type !== 'tool_result')
  const toolResults = b.filter(x => x.type === 'tool_result')
  if (toSmoosh.length === 0) {
    return [...a, ...b]
  }

  const smooshed = smooshIntoToolResult(lastBlock, toSmoosh)
  if (smooshed === null) {
    return [...a, ...b]
  }

  return [...a.slice(0, -1), smooshed, ...toolResults]
}
```

这个函数处理 **a 的尾是 tool_result + b 的某些 block 应该 smoosh 进去** 的情况。

注释（`messages.ts:2604-2609`）：

> See https://anthropic.slack.com/archives/C06FE2FP0Q2/p1747586370117479 and https://anthropic.slack.com/archives/C0AHK9P0129/p1773159663856279:
> any sibling after tool_result renders as `</function_results>\n\nHuman:<...>` on the wire. Repeated mid-conversation, this teaches capy to emit `Human:` at a bare tail → 3-token empty end_turn. A/B (sai-20260310-161901) validated: smoosh into tool_result.content → 92% → 0%.

这是 [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md) 里讲过的同一段注释。从消息管线的角度看：

- **不 smoosh**：tool_result 后面跟一个 text block，wire 上变成 `</function_results>\n\nHuman:<text>`，模型学到"Human:"作为停止信号；
- **smoosh**：把 text block 塞进 tool_result.content 里，wire 上没有 sibling，没问题。

代码走两条路径：

### 4.5.1 Legacy 路径（gate OFF）

只 smoosh "tool_result.content 是字符串 + b 全是 text" 的情况。这是 `tengu_chair_sermon` gate 上线前的旧逻辑。

### 4.5.2 Universal smoosh（gate ON）

**所有非 tool_result 的 block**（text / image / document / search_result）都 smoosh 进 tool_result.content；b 里如果有 tool_result block，作为 sibling 保留。

注意 `smoosh` 失败（返回 null）的情况——tool_reference 不能和别的 block 类型混。fallback 是不 smoosh，直接 concat。

## 4.6 底层折叠：`smooshIntoToolResult`

`messages.ts:2534-2598`，已经在 [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md#1.4.5) 详细拆过，这里复述要点：

- 输入：`tr: ToolResultBlockParam` + `blocks: ContentBlockParam[]`
- 输出：`ToolResultBlockParam | null`
- 边界：tool_reference 在 existing.content 里 → 返回 null；is_error 强制 text-only → 过滤非 text
- 公共路径：原 content 是 string + b 全 text → 输出 string（保留旧形状）；其它 → 输出数组并合并相邻 text

## 4.7 后处理：`smooshSystemReminderSiblings`

`messages.ts:1835-1873`，主循环 forEach 之后调一次（pass 6 的一部分）。

它处理一种**特殊情况**：经过 mergeAdjacent 后，user message content[] 里同时含 tool_result 和以 `<system-reminder>` 开头的 text block——把后者 smoosh 进**最后一个** tool_result。

为什么需要这个 pass 而不是在 mergeUserContentBlocks 里就处理？因为 SR-text 可能来自：

- attachment 转 user 后被 ensureSystemReminderWrap 包装的内容；
- PreToolUse hook 的 additionalContext；
- relocateToolReferenceSiblings 的输出；

这些 SR-text 在被合并时**还不知道有没有相邻的 tool_result**——可能在合并多步之后才出现并置。所以需要一个全局 pass 兜底。

注释（`messages.ts:1819-1832`）：

> Final pass: smoosh any `<system-reminder>`-prefixed text siblings into the last tool_result of the same user message. Catches siblings from:
> - PreToolUse hook additionalContext (Gap F)
> - relocateToolReferenceSiblings output (Gap E)
> - any attachment-origin text that escaped merge-time smoosh
>
> Non-system-reminder text (real user input, TOOL_REFERENCE_TURN_BOUNDARY, context-collapse `<collapsed>` summaries) stays untouched.

明确**只 smoosh `<system-reminder>` 开头的 text**——其它 text 不动。`TOOL_REFERENCE_TURN_BOUNDARY` 文本就是一个例外（它不以 SR 开头，所以保留为 sibling，这是设计意图——见 [05](./05-tool_reference与协议边界事故.md)）。

## 4.8 特殊版本：`mergeUserMessagesAndToolResults`

`messages.ts:2372-2387`：

```ts
export function mergeUserMessagesAndToolResults(
  a: UserMessage,
  b: UserMessage,
): UserMessage {
  const lastContent = normalizeUserTextContent(a.message.content)
  const currentContent = normalizeUserTextContent(b.message.content)
  return {
    ...a,
    message: {
      ...a.message,
      content: hoistToolResults(
        mergeUserContentBlocks(lastContent, currentContent),
      ),
    },
  }
}
```

attachment case 用的合并版本。和 `mergeUserMessages` 的区别：

| | mergeUserMessages | mergeUserMessagesAndToolResults |
|---|---|---|
| 接缝处理 | `joinTextAtSeam`（text-text 加 `\n`） | `mergeUserContentBlocks`（可能触发 smoosh） |
| uuid 处理 | `a.isMeta ? b.uuid : a.uuid` | 直接 `...a`（保留 a 的 uuid） |
| isMeta 合并 | HISTORY_SNIP 路径下有特殊规则 | 不动 |
| 用途 | 主循环 user case 合并 / mergeAdjacent | attachment case 合并 |

为什么 attachment 用不同版本？因为 attachment 的内容可能包含 tool_result 引用（如 `async_hook_response` 注入 toolUseID 关联的 hook 输出），需要走完整的 smoosh 逻辑。普通 user message 合并不需要触发 smoosh（SR-text 在后处理段单独走 `smooshSystemReminderSiblings`）。

## 4.9 同 ID assistant 合并：`mergeAssistantMessages`

`messages.ts:2389-2400`：

```ts
export function mergeAssistantMessages(
  a: AssistantMessage,
  b: AssistantMessage,
): AssistantMessage {
  return {
    ...a,
    message: {
      ...a.message,
      content: [...a.message.content, ...b.message.content],
    },
  }
}
```

极简——直接 concat content。

为什么这么简单？因为 assistant message 没有"接缝处 `\n` 问题"（API 不会拼接相邻 text 是 user message 的行为，assistant 不一样）；也没有 tool_result 需要 hoist；也没有 system-reminder 需要 smoosh。

实际上**这个合并几乎不发生**——只在 teammate 模式下，多个 agent 的流式响应交错到达、且它们共享 `message.id` 时才合并。绝大多数情况下，主 forEach 的查找循环找不到同 ID 的前置消息，直接 push。

## 4.10 整体流程：以一次 normalize 为例

假设输入 messages 是这样的（简化）：

```
[
  { type: 'user', content: [text("fix bug")] },
  { type: 'assistant', content: [tool_use(Read, ...)] },
  { type: 'user', content: [tool_result(...)] },              ← #1
  { type: 'attachment', attachment: { type: 'relevant_memories', ... } }, ← #2
  { type: 'user', content: [text("what about X")] },          ← #3
]
```

经过 forEach 处理：

```
Step 1: user "fix bug" → push
Step 2: assistant tool_use → push (无同 ID 前置)
Step 3: user tool_result → push (前面是 assistant)
Step 4: attachment relevant_memories:
        - normalizeAttachmentForAPI → [UserMessage(SR-wrapped memory)]
        - ensureSystemReminderWrap → 已 SR-wrap，noop
        - last 是 user (tool_result) → mergeUserMessagesAndToolResults
            → mergeUserContentBlocks
                → b 全是 text (SR-wrapped)
                → smooshIntoToolResult(lastBlock, SRText)
                → 返回 ToolResultBlockParam (内含 SR-text)
            → hoistToolResults
            → 结果: user with [tool_result(merged with SR-text)]
Step 5: user "what about X":
        - 应用各种 strip / inject
        - last 是 user → mergeUserMessages
            → joinTextAtSeam([tool_result], [text("what...")])
              tool_result 不是 text, 不需要加 \n → 直接 concat
            → hoistToolResults: [tool_result, text("what...")]
```

最终结果：

```
[
  { user, content: [text("fix bug")] },
  { assistant, content: [tool_use(Read, ...)] },
  { user, content: [tool_result(with SR-text inside), text("what about X")] },
]
```

注意最终只有 3 条 message——attachment 被吸收进 user，SR-text 被折叠进 tool_result.content，tool_result 被 hoist 到前面。

## 4.11 设计哲学总结

### 1. 单一职责的小函数

每个函数只做一件事：

- `joinTextAtSeam`：只处理 text-text 接缝的 `\n`；
- `hoistToolResults`：只重排 tool_result 到前面；
- `smooshIntoToolResult`：只折叠 block 进 tool_result.content；
- `mergeUserContentBlocks`：决定要不要 smoosh；
- `mergeUserMessages`：决定 uuid / isMeta 怎么传；
- `mergeAdjacentUserMessages`：决定哪些相邻 user 要合并。

每一层都简单。**复杂性藏在层级关系里，不藏在单函数里**。

### 2. 协议陷阱用结构 fix

API 的协议陷阱（tool_result 必须在前、相邻 text 不加分隔符）都用结构操作解决：hoist 重排、joinTextAtSeam 加 `\n`。不依赖运行时检查。

### 3. 模型行为陷阱用 smoosh fix

A/B 实验发现的"`Human:` 提前停止"问题用 smoosh 解决——把 sibling 文本塞进 tool_result.content。这是**用结构变换让协议层的副作用消失**。

### 4. uuid 保留策略服务于下游分类器

`a.isMeta ? b.uuid : a.uuid` 这种选择不是任意的——它服务于 [id:xxx] tag 的稳定性。**合并函数的契约不只是"内容正确"，还包括"派生标识符稳定"**。

### 5. feature gate 让旧逻辑保留

`tengu_chair_sermon` 控制两套合并策略（legacy 字符串 smoosh + universal block smoosh）。新策略上线后旧策略保留——支持快速回滚。

### 6. 每个函数都幂等

- `joinTextAtSeam` 重复跑只会加多余的 `\n`（但合并函数不会被同输入跑多次，这只是理论性）；
- `hoistToolResults` 重复跑结果不变；
- `smoosh` 在已经 smoosh 过的输入上是 no-op；
- `mergeAdjacent` 单次扫描，第二次跑没相邻 user 可合。

这让 normalize 整体可以反复跑而不破坏结果。

## 4.12 小结

- 合并族是 6 层嵌套的函数：mergeAdjacent → merge / mergeAndToolResults → mergeContentBlocks → smoosh / hoist / joinAtSeam；
- 每一层处理一个具体的协议或模型行为陷阱：
  - 连续 user → mergeAdjacent
  - text 接缝拼接 → joinTextAtSeam (`\n` 加在 a 末)
  - tool_result 必须在前 → hoistToolResults
  - SR-text 不能在 tool_result 后 → smoosh
  - tool_reference 不能混 → smoosh 返回 null fallback
- `mergeAssistantMessages` 极简，因为 assistant 没有这些约束（仅用于 teammate 流式交错）；
- 一个常被忽视的细节：`\n` 加在 a 的末尾不是 b 的开头——为了保 `startsWith('<system-reminder>')` 分类器；
- 单函数简单，复杂性在层级关系里——这种设计可读、可测、可灰度。

下一篇 → [05 tool_reference 与协议边界事故](./05-tool_reference与协议边界事故.md)
