# 03 normalize 主循环与三路分发

> `normalizeMessagesForAPI` 是整个管线的指挥中枢——一个 380+ 行的函数，把 `Message[]` 变成 API 能接受的 `(UserMessage | AssistantMessage)[]`。本篇逐段拆它的结构。

## 3.1 函数签名与定位

`messages.ts:1989-1992`：

```ts
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tools = [],
): (UserMessage | AssistantMessage)[] {
  // ...
}
```

两个输入：

- `messages`：当前会话的完整消息历史（包括 user/assistant/attachment/progress/system/tombstone）；
- `tools`：当前可用工具列表（用于过滤无效 tool_reference）。

返回 `(UserMessage | AssistantMessage)[]`——**只剩两种 type**。所有 attachment / progress / system / synthetic 都被转换或过滤掉。

调用点：claude.ts 在每次准备 API 请求时调一次。注意是**每次**——会话越长，每轮 normalize 的开销越大。这就是为什么 normalize 函数内做的事必须严格是"纯数据变换"——没有 IO，CPU 复杂度尽量线性。

## 3.2 函数结构鸟瞰

整个函数可以分成 **3 大段**：

```
┌─────────────────────────────────────────────────────────────┐
│ 段 1: 预处理 (lines 1994-2054)                              │
│   - 构建 availableToolNames                                 │
│   - reorderAttachmentsForAPI                                │
│   - filter virtual                                          │
│   - build errorToBlockTypes / stripTargets map              │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 段 2: 主 forEach + switch (lines 2056-2293)                │
│   过滤 progress / system(non-local) / synthetic-error       │
│   然后 forEach message:                                     │
│     case 'system' → 转 user                                 │
│     case 'user' → strip toolref / strip 错误源 / 注入       │
│                    TOOL_REFERENCE_TURN_BOUNDARY / merge prev│
│     case 'assistant' → normalize tool_use / merge same-id   │
│     case 'attachment' → expand / SR-wrap / merge prev       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 段 3: 后处理串联 (lines 2295-2369)                          │
│   - relocateToolReferenceSiblings (gate ON)                 │
│   - filterOrphanedThinkingOnlyMessages                      │
│   - filterTrailingThinkingFromLastAssistant                 │
│   - filterWhitespaceOnlyAssistantMessages                   │
│   - ensureNonEmptyAssistantContent                          │
│   - mergeAdjacent + smooshSystemReminderSiblings (gate ON)  │
│   - sanitizeErrorToolResultContent                          │
│   - appendMessageTagToUserMessage (HISTORY_SNIP)            │
│   - validateImagesForAPI                                    │
└─────────────────────────────────────────────────────────────┘
```

下面逐段拆。

## 3.3 段 1：预处理（lines 1994-2054）

### 3.3.1 收集可用工具名

```ts
const availableToolNames = new Set(tools.map(t => t.name))
```

供后面 `stripUnavailableToolReferencesFromUserMessage` 使用——用来过滤"指向已不存在工具"的 tool_reference block（如 MCP server 断开后，留下的 reference 要删掉）。

### 3.3.2 `reorderAttachmentsForAPI` —— attachment 上浮

```ts
const reorderedMessages = reorderAttachmentsForAPI(messages).filter(
  m => !((m.type === 'user' || m.type === 'assistant') && m.isVirtual),
)
```

第一步：调 `reorderAttachmentsForAPI`（`messages.ts:1481-1527`，未展开），它的作用是**把 attachment 消息往前移**——直到撞到 tool_result 或 assistant 消息为止。

为什么需要 reorder？因为 attachment 在 store 里按"产生时间"插入，但实际上它的"逻辑位置"应该是**紧贴它要修饰的那条用户消息**。比如：

```
[user: "fix the bug"]
[assistant: "let me read the file"]
[assistant: <tool_use Read>]
[user: <tool_result>]
[attachment: relevant_memories]    ← 应该贴在下一条 user 之前
[assistant: ...]
[user: "what about X?"]
```

reorder 后变成：

```
[user: "fix the bug"]
[assistant: "let me read the file"]
[assistant: <tool_use Read>]
[user: <tool_result>]
[assistant: ...]
[attachment: relevant_memories]    ← 移到这里
[user: "what about X?"]
```

这样 attachment 紧贴下一条 user message，normalize 时可以自然合并进去。

### 3.3.3 过滤 virtual

```ts
.filter(
  m => !((m.type === 'user' || m.type === 'assistant') && m.isVirtual),
)
```

`isVirtual: true` 的消息**永不进 API**。这是 UI 用的占位（如 REPL 内部的模拟工具调用）。

### 3.3.4 errorToBlockTypes —— 错误类型到 block 类型的映射

```ts
const errorToBlockTypes: Record<string, Set<string>> = {
  [getPdfTooLargeErrorMessage()]: new Set(['document']),
  [getPdfPasswordProtectedErrorMessage()]: new Set(['document']),
  [getPdfInvalidErrorMessage()]: new Set(['document']),
  [getImageTooLargeErrorMessage()]: new Set(['image']),
  [getRequestTooLargeErrorMessage()]: new Set(['document', 'image']),
}
```

5 种 API 错误的字符串 → "应该剥离哪种 content block"的映射。

例如：API 上次返回了 `pdf_too_large` 错误，那么生成这个错误的 user message 里的 `document` block 就应该在下次请求时**剥离掉**——否则每次重发都会 400。

### 3.3.5 stripTargets —— 反向追溯

```ts
const stripTargets = new Map<string, Set<string>>()
for (let i = 0; i < reorderedMessages.length; i++) {
  const msg = reorderedMessages[i]!
  if (!isSyntheticApiErrorMessage(msg)) {
    continue
  }
  // Determine which error this is
  const errorText =
    Array.isArray(msg.message.content) &&
    msg.message.content[0]?.type === 'text'
      ? msg.message.content[0].text
      : undefined
  if (!errorText) {
    continue
  }
  const blockTypesToStrip = errorToBlockTypes[errorText]
  if (!blockTypesToStrip) {
    continue
  }
  // Walk backward to find the nearest preceding isMeta user message
  for (let j = i - 1; j >= 0; j--) {
    const candidate = reorderedMessages[j]!
    if (candidate.type === 'user' && candidate.isMeta) {
      const existing = stripTargets.get(candidate.uuid)
      if (existing) {
        for (const t of blockTypesToStrip) {
          existing.add(t)
        }
      } else {
        stripTargets.set(candidate.uuid, new Set(blockTypesToStrip))
      }
      break
    }
    if (isSyntheticApiErrorMessage(candidate)) {
      continue
    }
    break
  }
}
```

这是个**反向追溯**逻辑：

1. 遍历每条消息，找出 `isSyntheticApiErrorMessage`（即上次 API 返回的错误占位）；
2. 看错误文本是哪种（pdf_too_large 等）；
3. 向前找最近的 `isMeta` user message——那就是**产生该错误的源**；
4. 记录"这条 user message 的 `<block_type>` 要被剥离"。

**为什么向前找 `isMeta`？** 因为 PDF/image 这种大附件通常通过 `@file.pdf` 这类 attachment 注入，附件转 user message 时带 `isMeta: true`。所以错误一定指向最近的 isMeta user message。

后面段 2 处理 user case 时会用 `stripTargets.get(uuid)` 决定要不要剥离这条消息的某些 block。

### 3.3.6 段 1 设计判断

这一段的精妙之处：

- **不修改原消息**：所有变换都生成新数据结构，原 messages 不动；
- **声明式错误恢复**：用 `errorToBlockTypes` 表声明"错误 → 修复动作"，加新错误只要加一行表项；
- **两遍扫描**：先建 stripTargets map，再单遍 forEach 应用——避免内嵌循环导致 O(n²)。

## 3.4 段 2：主 forEach + switch（lines 2056-2293）

### 3.4.1 入口过滤

```ts
const result: (UserMessage | AssistantMessage)[] = []
reorderedMessages
  .filter(
    (
      _,
    ): _ is
      | UserMessage
      | AssistantMessage
      | AttachmentMessage
      | SystemLocalCommandMessage => {
      if (
        _.type === 'progress' ||
        (_.type === 'system' && !isSystemLocalCommandMessage(_)) ||
        isSyntheticApiErrorMessage(_)
      ) {
        return false
      }
      return true
    },
  )
  .forEach(message => { /* switch 在这里 */ })
```

filter 三种消息：

- `progress`：UI 进度消息，不发 API；
- 非 `local_command` 的 `system` 消息：UI 事件（如错误提示），不发 API；
- `isSyntheticApiErrorMessage`：上次 API 错误的占位（前面已经用来构建 stripTargets，现在过滤掉）。

剩下的 forEach 处理：`user` / `assistant` / `attachment` / `system(local_command)`。

### 3.4.2 `case 'system'` —— 转 user

```ts
case 'system': {
  const userMsg = createUserMessage({
    content: message.content,
    uuid: message.uuid,
    timestamp: message.timestamp,
  })
  const lastMessage = last(result)
  if (lastMessage?.type === 'user') {
    result[result.length - 1] = mergeUserMessages(lastMessage, userMsg)
    return
  }
  result.push(userMsg)
  return
}
```

只有 `local_command` 子类型的 system 消息会到这里（filter 已经过滤了其它 system）。把它转成 user message——这样 `/bash echo hi` 的输出能让模型在后续轮次看到。

注释（`messages.ts:2079-2080`）：

> local_command system messages need to be included as user messages so the model can reference previous command output in later turns

转完后**优先合并**到前一条 user。

### 3.4.3 `case 'user'` —— 最复杂的分支

```ts
case 'user': {
  // 1. strip tool_reference (按 tool-search 是否启用走不同分支)
  let normalizedMessage = message
  if (!isToolSearchEnabledOptimistic()) {
    normalizedMessage = stripToolReferenceBlocksFromUserMessage(message)
  } else {
    normalizedMessage = stripUnavailableToolReferencesFromUserMessage(
      message,
      availableToolNames,
    )
  }

  // 2. 应用 stripTargets (移除引发上次错误的 document/image block)
  const typesToStrip = stripTargets.get(normalizedMessage.uuid)
  if (typesToStrip && normalizedMessage.isMeta) {
    const content = normalizedMessage.message.content
    if (Array.isArray(content)) {
      const filtered = content.filter(
        block => !typesToStrip.has(block.type),
      )
      if (filtered.length === 0) {
        return  // 整条消息被剥光了，跳过
      }
      if (filtered.length < content.length) {
        normalizedMessage = { ...normalizedMessage, ... }
      }
    }
  }

  // 3. 注入 TOOL_REFERENCE_TURN_BOUNDARY (gate OFF 时的 #21049 fallback)
  if (!checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_toolref_defer_j8m')) {
    if (
      contentHasToolReference(contentAfterStrip) &&
      !contentAfterStrip.some(b => b.type === 'text' && b.text.startsWith(TOOL_REFERENCE_TURN_BOUNDARY))
    ) {
      normalizedMessage = { ..., content: [...contentAfterStrip, { type: 'text', text: TOOL_REFERENCE_TURN_BOUNDARY }] }
    }
  }

  // 4. 合并到前一条 user
  const lastMessage = last(result)
  if (lastMessage?.type === 'user') {
    result[result.length - 1] = mergeUserMessages(lastMessage, normalizedMessage)
    return
  }
  result.push(normalizedMessage)
  return
}
```

4 个动作按顺序：

**(1) strip tool_reference**

- tool-search 未启用：剥离**所有** `tool_reference` block（beta-only 不能发普通 API）；
- tool-search 启用：只剥离**指向已不存在工具**的 reference（如 MCP 断了）。

**(2) 应用 stripTargets**

如果这条 isMeta user 在 stripTargets 里有记录，剥离指定 block 类型。`document` 错误 → 剥 document，`image` 错误 → 剥 image。整条剥光就 return（跳过该消息）。

**(3) 注入 TOOL_REFERENCE_TURN_BOUNDARY**

只在 `tengu_toolref_defer_j8m` gate OFF 时执行。这是 #21049 事故的**预防方式 fallback**——详见 [05](./05-tool_reference与协议边界事故.md)。

**(4) 合并 / 推入**

按规则 push 或 merge。merge 的细节在 [04](./04-合并smoosh与hoist三连击.md)。

### 3.4.4 `case 'assistant'` —— tool_use 输入标准化 + 同 ID 合并

```ts
case 'assistant': {
  const toolSearchEnabled = isToolSearchEnabledOptimistic()
  const normalizedMessage: AssistantMessage = {
    ...message,
    message: {
      ...message.message,
      content: message.message.content.map(block => {
        if (block.type === 'tool_use') {
          const tool = tools.find(t => toolMatchesName(t, block.name))
          const normalizedInput = tool
            ? normalizeToolInputForAPI(tool, block.input as Record<string, unknown>)
            : block.input
          const canonicalName = tool?.name ?? block.name

          if (toolSearchEnabled) {
            // 保留 'caller' 等 tool-search 字段
            return { ...block, name: canonicalName, input: normalizedInput }
          }

          // 显式构造只含标准 API 字段的 tool_use block
          return {
            type: 'tool_use' as const,
            id: block.id,
            name: canonicalName,
            input: normalizedInput,
          }
        }
        return block
      }),
    },
  }

  // Find a previous assistant message with the same message ID and merge.
  for (let i = result.length - 1; i >= 0; i--) {
    const msg = result[i]!
    if (msg.type !== 'assistant' && !isToolResultMessage(msg)) {
      break
    }
    if (msg.type === 'assistant') {
      if (msg.message.id === normalizedMessage.message.id) {
        result[i] = mergeAssistantMessages(msg, normalizedMessage)
        return
      }
      continue
    }
  }
  result.push(normalizedMessage)
  return
}
```

两个动作：

**(1) tool_use 输入标准化 + 字段剥离**

- 找到对应工具，调 `normalizeToolInputForAPI` 让工具有机会修正 input（比如 ExitPlanModeV2 会 strip 内部的 `plan` 字段，因为 API 不需要它）；
- canonicalName：处理 MCP 的命名前缀差异；
- tool-search 启用时保留所有字段（包括 `caller`）；
- 未启用时**显式构造**只含标准 API 字段的 block——这是为了防止旧 session 里持久化的 `caller` 字段被无意中发出去。

**(2) 同 message ID 合并**

向后扫描已处理的 result，找到 message ID 相同的 assistant message 就 merge。注释（`messages.ts:2247-2249`）说：

> concurrent agents (teammates) can interleave streaming content blocks from multiple API responses with different message IDs.

teammate 模式下，多个 agent 的流式响应可能交错到达——它们用 `message.id` 区分，相同 ID 的合并到一起。

**关键扫描逻辑**：从 result 末尾往前找，只跨越 assistant 消息和 tool_result 消息——遇到其它 user message 就停止。这避免合并跨越用户输入的 message。

### 3.4.5 `case 'attachment'` —— 展开 + SR wrap + 合并

```ts
case 'attachment': {
  const rawAttachmentMessage = normalizeAttachmentForAPI(message.attachment)
  const attachmentMessage = checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_chair_sermon')
    ? rawAttachmentMessage.map(ensureSystemReminderWrap)
    : rawAttachmentMessage

  const lastMessage = last(result)
  if (lastMessage?.type === 'user') {
    result[result.length - 1] = attachmentMessage.reduce(
      (p, c) => mergeUserMessagesAndToolResults(p, c),
      lastMessage,
    )
    return
  }

  result.push(...attachmentMessage)
  return
}
```

3 步：

**(1) `normalizeAttachmentForAPI`**：把单个 attachment 转换成 `UserMessage[]`（注意是数组——某些 attachment 可能展开成多条消息，如 directory listing 会展开成 caveat + command + stdout 三条）。详见 [07](./07-attachment-normalizer.md)。

**(2) `ensureSystemReminderWrap`**（gate ON 时）：保证所有 attachment-origin 文本都以 `<system-reminder>` 包装。幂等。

**(3) 合并到前一条 user**：用 `mergeUserMessagesAndToolResults` 把展开出来的多条 user 全部 reduce 进前一条 user（或推入新消息）。

## 3.5 段 3：后处理串联（lines 2295-2369）

主 forEach 之后是 8 个串联的 pass。**每一个 pass 都对应一个具体的 API 错误或事故修复**：

```ts
// 1. relocateToolReferenceSiblings (gate ON 路径)
const relocated = checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_toolref_defer_j8m')
  ? relocateToolReferenceSiblings(result)
  : result

// 2. 移除孤立的 thinking-only 消息 (compact 切割可能产生)
const withFilteredOrphans = filterOrphanedThinkingOnlyMessages(relocated)

// 3. 移除最后一条 assistant 末尾的 thinking
const withFilteredThinking =
  filterTrailingThinkingFromLastAssistant(withFilteredOrphans)

// 4. 移除全空白 assistant 消息
const withFilteredWhitespace =
  filterWhitespaceOnlyAssistantMessages(withFilteredThinking)

// 5. 确保 assistant content[] 非空 (空就补一个默认 text)
const withNonEmpty = ensureNonEmptyAssistantContent(withFilteredWhitespace)

// 6. mergeAdjacent + smoosh
const smooshed = checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_chair_sermon')
  ? smooshSystemReminderSiblings(mergeAdjacentUserMessages(withNonEmpty))
  : withNonEmpty

// 7. 修复 is_error tool_result 含非 text block 的问题
const sanitized = sanitizeErrorToolResultContent(smooshed)

// 8. HISTORY_SNIP: 给每条 user 加 [id:xxx] 标签
if (feature('HISTORY_SNIP') && process.env.NODE_ENV !== 'test') { ... }

// 9. 图像校验
validateImagesForAPI(sanitized)
```

每个 pass 的细节在后面几篇展开：

- pass 1（relocate）→ [05](./05-tool_reference与协议边界事故.md)
- pass 2-5（filter / ensureNonEmpty）→ [06](./06-协议合规过滤层.md)
- pass 6（merge + smoosh）→ [04](./04-合并smoosh与hoist三连击.md)
- pass 7（sanitizeError）→ [06](./06-协议合规过滤层.md)
- pass 8（HISTORY_SNIP tag）→ 简单说明在本节后面

### 3.5.1 pass 顺序敏感性

`messages.ts:2313-2320` 注释专门讨论了 pass 顺序：

> Order matters: strip trailing thinking first, THEN filter whitespace-only messages. The reverse order has a bug: a message like `[text("\n\n"), thinking("...")]` survives the whitespace filter (has a non-text block), then thinking stripping removes the thinking block, leaving `[text("\n\n")]` — which the API rejects.
>
> These multi-pass normalizations are inherently fragile — each pass can create conditions a prior pass was meant to handle. Consider unifying into a single pass that cleans content, then validates in one shot.

逐字翻译：

- 顺序重要：先 strip trailing thinking，**然后**过滤 whitespace-only；
- 反过来有 bug：`[text("\n\n"), thinking("...")]` 这条消息——
  - whitespace filter 看：有非 text block（thinking），不过滤；
  - 然后 strip thinking：剩 `[text("\n\n")]`；
  - 发 API：被拒绝（全空白 + content[].length=1）。
- **作者自己说**：这种多 pass 的设计天然脆弱，每个 pass 都可能制造另一个 pass 本来要处理的情况。建议未来合并成单 pass 清洗 + 单 pass 验证。

这是个**罕见的**"代码自己批评自己"——能在工业代码里看到这种诚实的注释很难得。这种"建议未来合并"的注释也提示我们：现在的设计是历史演化的产物，不是一开始就这么复杂。

### 3.5.2 pass 8: HISTORY_SNIP 标签

```ts
if (feature('HISTORY_SNIP') && process.env.NODE_ENV !== 'test') {
  const { isSnipRuntimeEnabled } = require('../services/compact/snipCompact.js')
  if (isSnipRuntimeEnabled()) {
    for (let i = 0; i < sanitized.length; i++) {
      if (sanitized[i]!.type === 'user') {
        sanitized[i] = appendMessageTagToUserMessage(
          sanitized[i] as UserMessage,
        )
      }
    }
  }
}
```

HISTORY_SNIP 是新的压缩策略——它需要让模型在历史里**通过 [id:xxx] 标签引用具体消息**来做 snip 操作。这里给每条 user 消息追加一个 `[id:xxx]` 标签（基于 uuid 派生）。

测试环境跳过——注释说这会改变消息内容 hash，破坏 VCR fixture 查找。

### 3.5.3 最后：validateImagesForAPI

```ts
validateImagesForAPI(sanitized)
return sanitized
```

最后一道闸门：扫描所有 image block 检查尺寸是否符合 API 限制（< 5MB、< 8000x8000 px）。失败抛错——上层会捕获并合成对应的 SyntheticApiErrorMessage，下一轮 normalize 时就走 stripTargets 路径剥离掉。

形成一个**闭环**：本轮检测出超大图像 → 本轮抛错 → 上层吃错合成 syntheticError → 下一轮 normalize 走 stripTargets 剥离 image → 之后请求都不再带这张图。

## 3.6 三路分发对照表

把 user / assistant / attachment 三个 case 横向对比：

| 维度 | user | assistant | attachment |
|---|---|---|---|
| 入口标准化 | strip tool_reference / strip 错误源 block | normalize tool_use input / 剥离 caller | 调 normalizeAttachmentForAPI 展开 |
| 协议陷阱处理 | 注入 TOOL_REFERENCE_TURN_BOUNDARY (gate OFF) | — | ensureSystemReminderWrap (gate ON) |
| 合并策略 | merge 前一条 user | merge 同 message.id 的 assistant | reduce 进前一条 user |
| 合并函数 | mergeUserMessages | mergeAssistantMessages | mergeUserMessagesAndToolResults |
| 失败处理 | 跳过空消息 | 直接 push (合并失败) | 多条全部 push |

可以看到：

- **user/attachment 都倾向合并到前一条 user**——因为 API 不允许连续 user message（Bedrock）；
- **assistant 倾向合并到相同 message ID 的 assistant**——支持 teammate 模式下交错流式；
- 三者各有自己的协议陷阱要处理：user 担心 tool_reference，attachment 担心标签包装，assistant 担心 caller 字段泄漏。

## 3.7 设计哲学回顾

读完整个主循环，可以抽几条判断：

### 1. 用表驱动的错误恢复

`errorToBlockTypes` 表 + 反向追溯 stripTargets——这种"声明式错误 → 修复"模式让加新错误类型只需要加一行表项。

### 2. 多 pass 是历史负担，作者承认

注释里直接说"建议未来合并"。这种工业代码里看到的诚实自批评是宝贵的——它告诉读者"这里不是最优设计，是渐进演化的痕迹"。

### 3. 三类消息共用一个 forEach，按 type 分发

如果用 4 个独立函数（normalizeUserMessages, normalizeAssistantMessages, ...）也能写，但放在一个 switch 里的好处：

- 共享前面构建的 stripTargets / availableToolNames；
- 共享后面的合并尾逻辑（push or merge prev）；
- 维持消息**位置敏感**——一条 user 可能要 merge 它前一条 user，一条 attachment 可能要 reduce 进前一条 user，必须在同一遍 forEach 里才能保持顺序。

### 4. feature gate 决定不同路径

`tengu_toolref_defer_j8m`、`tengu_chair_sermon`、`HISTORY_SNIP` 三个 gate 在同一函数里有三处决策——支持灰度发布。这种"同一管线、不同 gate 路径"的设计让 A/B 实验可以无缝接入。

### 5. validate 在最后，让闭环可达

`validateImagesForAPI` 在 return 前最后跑——它抛错会让本次请求失败，但**抛错信息会被上层捕获并合成 syntheticError 加进消息历史**，下一轮 normalize 自动剥离问题源。

这是一个把"错误处理"也当成"数据流"的设计：错误不是异常路径，而是数据流的一个分支。

## 3.8 小结

- `normalizeMessagesForAPI` 380+ 行分 3 段：预处理 / 主 forEach / 后处理串联；
- 预处理建立**反向追溯**：扫描错误占位 → 找到错误源 → 标记要剥离的 block；
- 主 forEach 按 type 分发到 4 个 case（system / user / assistant / attachment），每个 case 都做"入口标准化 + 协议陷阱处理 + 合并/推入"；
- 后处理 9 个 pass 串联，每个对应一类协议陷阱；pass 顺序敏感，作者承认设计脆弱；
- HISTORY_SNIP 标签给每条 user 加 [id:xxx]，支持新压缩策略；
- `validateImagesForAPI` 闭环：本轮失败 → 合成 syntheticError → 下轮自动剥离。

下一篇 → [04 合并 smoosh 与 hoist 三连击](./04-合并smoosh与hoist三连击.md)
