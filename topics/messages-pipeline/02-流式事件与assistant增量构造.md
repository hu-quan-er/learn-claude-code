# 02 流式事件与 assistant 增量构造

> assistant 消息不是用 `createAssistantMessage` 构造的——是从 Anthropic Messages API 的 SSE 流式事件**一段一段累积**出来的。本篇拆 `handleMessageFromStream` 这个中央分发器，看流式事件如何变成最终消息。

## 2.1 为什么需要流式

API 不是"发完请求等完整回复返回"的同步模型，而是 SSE 流——服务器边生成边吐字节。Claude Code 利用这一点做三件事：

1. **UI 实时显示**：模型刚开始吐字符就显示，不等整段生成完——大幅降低用户感知延迟；
2. **超大输出降存峰值**：流式可以边接收边处理，不用一次性把整段加载进内存；
3. **早停（abort）**：用户按 ESC 时可以立刻中断 fetch，不浪费已经产生但还没传完的 token。

代价是：**收到的不是完整的 message，而是一连串增量事件**。Claude Code 必须能：

- 边接收边构造 `AssistantMessage`；
- 在每次事件上更新 UI 状态（`onSetStreamMode` 让 UI 显示"requesting / thinking / responding / tool-input / tool-use" 几种状态）；
- 在事件结束后把累积出的 message 提交到 store 走 normalize 流程。

`handleMessageFromStream` 就是这个分发器。

## 2.2 SSE 事件类型清单

Anthropic Messages API 的 SSE 流大致包含这些事件：

| 事件类型 | 含义 |
|---|---|
| `stream_request_start` | Claude Code 自己合成的（不是 API 的），标记请求 fetch 开始 |
| `message_start` | API 通知：开始流式响应。带 `usage` 初值和 model 信息 |
| `content_block_start` | 开始一个新的 content block。block 可能是 `thinking` / `text` / `tool_use` / 各种 server-side tool result 等 |
| `content_block_delta` | content block 的增量内容。delta 类型决定 payload 是什么（`text_delta` / `input_json_delta` / `thinking_delta` / `signature_delta`） |
| `content_block_stop` | 当前 content block 结束 |
| `message_delta` | message 级别的增量（如 stop_reason 已确定） |
| `message_stop` | message 整体结束 |

Claude Code 还有几种内部事件类型：

| 类型 | 用途 |
|---|---|
| `tombstone` | "撤回"标记，删掉之前 emit 的某条消息 |
| `tool_use_summary` | SDK 专用的工具调用摘要 |
| `Message` (user/assistant/...) | 完整的、非流式的消息（如 API 返回时一次性给出整段，或者本地构造的消息） |

`handleMessageFromStream` 同时处理这两类事件——既要处理流式 delta，也要处理非流式的完整 Message。

## 2.3 `handleMessageFromStream` 主入口

`messages.ts:2930-3094`：

```ts
export function handleMessageFromStream(
  message:
    | Message
    | TombstoneMessage
    | StreamEvent
    | RequestStartEvent
    | ToolUseSummaryMessage,
  onMessage: (message: Message) => void,
  onUpdateLength: (newContent: string) => void,
  onSetStreamMode: (mode: SpinnerMode) => void,
  onStreamingToolUses: (
    f: (streamingToolUse: StreamingToolUse[]) => StreamingToolUse[],
  ) => void,
  onTombstone?: (message: Message) => void,
  onStreamingThinking?: (
    f: (current: StreamingThinking | null) => StreamingThinking | null,
  ) => void,
  onApiMetrics?: (metrics: { ttftMs: number }) => void,
  onStreamingText?: (f: (current: string | null) => string | null) => void,
): void {
  // ...
}
```

它的接口设计是**回调式**——不返回数据，而是通过 8 个回调把不同事件的副作用 dispatch 出去：

| 回调 | 触发时机 | 上游做什么 |
|---|---|---|
| `onMessage` | 完整 Message 到达（非流式 / 累积完成） | 加进 store |
| `onUpdateLength` | 任何 delta 增量内容 | 计算 OTPS（output tokens per second）指标、更新 UI 进度条 |
| `onSetStreamMode` | 状态切换 | 切换 spinner 类型（"thinking" → 旋转思考 icon，"responding" → 流式文字 icon 等） |
| `onStreamingToolUses` | tool_use 块开始 / input JSON 增量 | UI 维护"正在构造的工具调用"列表 |
| `onTombstone` | tombstone 事件 | 从消息列表里删掉指定消息 |
| `onStreamingThinking` | thinking 块开始/完成 | UI 实时显示思考内容 |
| `onApiMetrics` | message_start 自带 ttftMs | 上报"首 token 时间"指标 |
| `onStreamingText` | text 块增量 | UI 实时显示文本，与最终 Message 之间做原子切换 |

这种"事件 → 回调"的设计让 `handleMessageFromStream` 自身**不持有状态**，所有状态由调用方持有（react useState、store reducer 等），它只是个 dispatcher。

## 2.4 分发逻辑逐段拆

### 2.4.1 入口分流：流式事件 vs 完整 Message

`messages.ts:2950-2982`：

```ts
if (
  message.type !== 'stream_event' &&
  message.type !== 'stream_request_start'
) {
  // Handle tombstone messages - remove the targeted message instead of adding
  if (message.type === 'tombstone') {
    onTombstone?.(message.message)
    return
  }
  // Tool use summary messages are SDK-only, ignore them in stream handling
  if (message.type === 'tool_use_summary') {
    return
  }
  // Capture complete thinking blocks for real-time display in transcript mode
  if (message.type === 'assistant') {
    const thinkingBlock = message.message.content.find(
      block => block.type === 'thinking',
    )
    if (thinkingBlock && thinkingBlock.type === 'thinking') {
      onStreamingThinking?.(() => ({
        thinking: thinkingBlock.thinking,
        isStreaming: false,
        streamingEndedAt: Date.now(),
      }))
    }
  }
  // Clear streaming text NOW so the render can switch displayedMessages
  // from deferredMessages to messages in the same batch, making the
  // transition from streaming text → final message atomic (no gap, no duplication).
  onStreamingText?.(() => null)
  onMessage(message)
  return
}
```

这段处理"非流式"事件：

- **tombstone**：删除前面发过的消息，调 `onTombstone`；
- **tool_use_summary**：SDK 专用，忽略；
- **完整 assistant**（resume 等场景）：提取 thinking block 用于 transcript 显示；
- **其他完整 Message**：直接 `onMessage` 加进 store。

关键设计：**`onStreamingText?.(() => null)` 在 onMessage 之前**清空流式文本——注释说"同一次 render batch 里把 displayedMessages 从 deferredMessages 切到 messages，避免出现'流式文本'和'最终消息'同时显示的瞬间"。这是 UI 渲染的原子性考虑。

### 2.4.2 `stream_request_start` —— 请求开始

```ts
if (message.type === 'stream_request_start') {
  onSetStreamMode('requesting')
  return
}
```

只做一件事：把 UI spinner 状态切到 `requesting`（"正在请求中"）。

### 2.4.3 `message_start` —— 服务端开始响应

```ts
if (message.event.type === 'message_start') {
  if (message.ttftMs != null) {
    onApiMetrics?.({ ttftMs: message.ttftMs })
  }
}
```

上报"首 token 时间"指标。`ttftMs` 是 time-to-first-token，由 fetch 层在收到第一个事件时计算并附加。这是观测性能用的重要指标。

注意没有 `return`——`message_start` 之后会继续走 switch（但 `message_start` 不在 switch 的 case 列表里，所以会走 default）。

### 2.4.4 `message_stop` —— 响应结束

```ts
if (message.event.type === 'message_stop') {
  onSetStreamMode('tool-use')
  onStreamingToolUses(() => [])
  return
}
```

切到 `tool-use` 状态（"开始执行工具"），清空 streaming tool uses 列表。

为什么是 `tool-use`？因为 message_stop 后，如果模型返回了 tool_use block，接下来就要开始执行工具——切到这个状态让 UI 显示"准备执行工具"。如果没有 tool_use，那 query loop 会很快进入下一轮或终止，状态会被覆盖。

### 2.4.5 `content_block_start` —— 块开始（多类型分流）

```ts
case 'content_block_start':
  onStreamingText?.(() => null)
  if (
    feature('CONNECTOR_TEXT') &&
    isConnectorTextBlock(message.event.content_block)
  ) {
    onSetStreamMode('responding')
    return
  }
  switch (message.event.content_block.type) {
    case 'thinking':
    case 'redacted_thinking':
      onSetStreamMode('thinking')
      return
    case 'text':
      onSetStreamMode('responding')
      return
    case 'tool_use': {
      onSetStreamMode('tool-input')
      const contentBlock = message.event.content_block
      const index = message.event.index
      onStreamingToolUses(_ => [
        ..._,
        {
          index,
          contentBlock,
          unparsedToolInput: '',
        },
      ])
      return
    }
    case 'server_tool_use':
    case 'web_search_tool_result':
    case 'code_execution_tool_result':
    case 'mcp_tool_use':
    case 'mcp_tool_result':
    case 'container_upload':
    case 'web_fetch_tool_result':
    case 'bash_code_execution_tool_result':
    case 'text_editor_code_execution_tool_result':
    case 'tool_search_tool_result':
    case 'compaction':
      onSetStreamMode('tool-input')
      return
  }
  return
```

按 content_block 的类型分流——每种 block 类型对应不同的 UI 状态：

| Block 类型 | 状态 |
|---|---|
| `thinking` / `redacted_thinking` | `thinking`（思考中） |
| `text` | `responding`（回答中） |
| `tool_use` | `tool-input`（构造工具输入） |
| `server_tool_use` / `mcp_tool_use` / 各种 server-side tool result / `compaction` | `tool-input` |

注意 `tool_use` case 还多做一件事：**在 streaming tool uses 列表里添加一个新条目**：

```ts
onStreamingToolUses(_ => [
  ..._,
  {
    index,
    contentBlock,
    unparsedToolInput: '',
  },
])
```

`index` 是 content block 的下标，`unparsedToolInput: ''` 是空字符串——因为 tool_use 的 input 是 JSON 字符串，需要边接收边累加（见下一节）。

### 2.4.6 `content_block_delta` —— 4 类增量

```ts
case 'content_block_delta':
  switch (message.event.delta.type) {
    case 'text_delta': {
      const deltaText = message.event.delta.text
      onUpdateLength(deltaText)
      onStreamingText?.(text => (text ?? '') + deltaText)
      return
    }
    case 'input_json_delta': {
      const delta = message.event.delta.partial_json
      const index = message.event.index
      onUpdateLength(delta)
      onStreamingToolUses(_ => {
        const element = _.find(_ => _.index === index)
        if (!element) {
          return _
        }
        return [
          ..._.filter(_ => _ !== element),
          {
            ...element,
            unparsedToolInput: element.unparsedToolInput + delta,
          },
        ]
      })
      return
    }
    case 'thinking_delta':
      onUpdateLength(message.event.delta.thinking)
      return
    case 'signature_delta':
      // Signatures are cryptographic authentication strings, not model
      // output. Excluding them from onUpdateLength prevents them from
      // inflating the OTPS metric and the animated token counter.
      return
    default:
      return
  }
```

4 种 delta：

#### text_delta —— 文本增量

```ts
const deltaText = message.event.delta.text
onUpdateLength(deltaText)
onStreamingText?.(text => (text ?? '') + deltaText)
```

调两个回调：

- `onUpdateLength(deltaText)`：让上层（OTPS metric / token counter）累计这一段；
- `onStreamingText`：把增量追加到当前流式文本——UI 实时显示。

#### input_json_delta —— tool_use 输入字符串增量

```ts
const delta = message.event.delta.partial_json
const index = message.event.index
onUpdateLength(delta)
onStreamingToolUses(_ => {
  const element = _.find(_ => _.index === index)
  if (!element) {
    return _
  }
  return [
    ..._.filter(_ => _ !== element),
    {
      ...element,
      unparsedToolInput: element.unparsedToolInput + delta,
    },
  ]
})
```

按 `index` 找到对应的 streaming tool use 条目，**追加 partial JSON**。

关键点：**这里只累加字符串，不尝试 parse**——因为 partial JSON 不是合法 JSON（可能括号都没闭合）。真正的 JSON parse 发生在 `normalizeContentFromAPI` 阶段（API 调用完成后，2660-2697 行）：

```ts
// messages.ts:2676-2697
if (typeof contentBlock.input === 'string') {
  const parsed = safeParseJSON(contentBlock.input)
  if (parsed === null && contentBlock.input.length > 0) {
    logEvent('tengu_tool_input_json_parse_fail', {
      toolName: sanitizeToolNameForAnalytics(contentBlock.name),
      inputLen: contentBlock.input.length,
    })
    if (process.env.USER_TYPE === 'ant') {
      logForDebugging(
        `tool input JSON parse fail: ${contentBlock.input.slice(0, 200)}`,
        { level: 'warn' },
      )
    }
  }
  normalizedInput = parsed ?? {}
}
```

如果整段累积完后 parse 失败：

- 上报 `tengu_tool_input_json_parse_fail` analytics 事件；
- ant 用户额外写 debug log（含前 200 字符的载荷）；
- **回退到空对象 `{}`**——让下游验证 tool input 时看到空，自然会失败并提示模型重试。

这是一个非常务实的 fallback 设计：**不让 JSON parse 失败阻塞整个对话**，而是让模型在下一轮看到"工具调用没参数"的结果，自己纠正。

#### thinking_delta —— 思考增量

```ts
case 'thinking_delta':
  onUpdateLength(message.event.delta.thinking)
  return
```

只上报长度，不维护流式状态（thinking 由非流式的完整 assistant message 上的 onStreamingThinking 处理）。

#### signature_delta —— 签名增量（特殊处理）

```ts
case 'signature_delta':
  // Signatures are cryptographic authentication strings, not model
  // output. Excluding them from onUpdateLength prevents them from
  // inflating the OTPS metric and the animated token counter.
  return
```

注释明确：thinking block 有时带签名（用来防止伪造 / 验证完整性）——这些**不是模型输出**，不应该计入 OTPS 指标和 token counter。

这是一个常被忽视的小细节：**指标不该被基础设施 token 污染**。如果把签名也算进 OTPS，会让"模型有效输出速度"被噪声拉低，看起来比实际慢。

### 2.4.7 其它事件

```ts
case 'content_block_stop':
  return
case 'message_delta':
  onSetStreamMode('responding')
  return
default:
  onSetStreamMode('responding')
  return
```

- `content_block_stop`：当前块结束，不做事（streaming tool use 的 unparsedToolInput 已经累积完成，等 normalizeContentFromAPI 收尾）；
- `message_delta`：通常带 `stop_reason` / `stop_sequence` 等元信息，切到 responding；
- `default`：未知事件类型，保守地切 responding。

## 2.5 状态机：SpinnerMode 切换图

整个流程里 `onSetStreamMode` 被调用 6+ 次，构成一个 UI 状态机：

```
                            (initial)
                                │
                                ▼
                       ╔════════════════╗
                       ║   requesting   ║  ← stream_request_start
                       ╚════════╤═══════╝
                                │
                                ▼
                                            (按 content_block.type 分流)
                       ┌────────┴────────┬───────────────────┐
                       │                 │                   │
                       ▼                 ▼                   ▼
                 ╔══════════╗     ╔═══════════╗      ╔═══════════╗
                 ║ thinking ║     ║ responding║      ║ tool-input║
                 ╚════╤═════╝     ╚═════╤═════╝      ╚═════╤═════╝
                      │                 │                  │
                      │ thinking_delta  │ text_delta       │ input_json_delta
                      │ (loop)          │ (loop)           │ (loop)
                      │                 │                  │
                      ▼                 ▼                  ▼
                 (content_block_stop, 进入下一个 block 或 message_stop)
                                        │
                                        ▼
                                ╔═══════════════╗
                                ║   tool-use    ║  ← message_stop
                                ╚═══════════════╝
                                        │
                                        ▼
                              (query loop 处理工具调用)
```

状态切换不是"原子单向"——一个 message 里可以有多个 content_block，每个 block 都会触发一次状态切换。例如典型的对话：

```
模型先 thinking → 再 text 回答 → 再 tool_use 调工具
状态序列：requesting → thinking → responding → tool-input → tool-use
```

UI 据此渲染不同的 spinner icon 和提示文本，给用户清晰的"模型在干什么"的反馈。

## 2.6 `normalizeContentFromAPI` —— 回流方向的标准化

handleMessageFromStream 之后，当**完整的 assistant message** 累积出来，会调 `normalizeContentFromAPI`（`messages.ts:2651-2751`）做收尾标准化。

这是 normalize 管线的**反方向**：normalizeMessagesForAPI 是"内部 Message → 发 API"，normalizeContentFromAPI 是"API 回流 → 内部 Message"。

它处理 5 类 content block：

### tool_use —— input JSON 解析

```ts
case 'tool_use': {
  if (
    typeof contentBlock.input !== 'string' &&
    !isObject(contentBlock.input)
  ) {
    throw new Error('Tool use input must be a string or object')
  }

  let normalizedInput: unknown
  if (typeof contentBlock.input === 'string') {
    const parsed = safeParseJSON(contentBlock.input)
    if (parsed === null && contentBlock.input.length > 0) {
      logEvent('tengu_tool_input_json_parse_fail', { ... })
    }
    normalizedInput = parsed ?? {}
  } else {
    normalizedInput = contentBlock.input
  }

  if (typeof normalizedInput === 'object' && normalizedInput !== null) {
    const tool = findToolByName(tools, contentBlock.name)
    if (tool) {
      try {
        normalizedInput = normalizeToolInput(
          tool,
          normalizedInput as { [key: string]: unknown },
          agentId,
        )
      } catch (error) {
        logError(new Error('Error normalizing tool input: ' + error))
      }
    }
  }

  return {
    ...contentBlock,
    input: normalizedInput,
  }
}
```

两步：

1. **JSON parse**：streaming 时累积的 partial_json 字符串，这里一次性 parse 成对象。失败上报并回退 `{}`；
2. **工具特定 corrections**：找到对应工具，调 `normalizeToolInput` 让工具有机会修正自己的 input（如类型转换、字段补全）。

关键注释（line 2670-2674）：

> The API has strange behaviour, where it returns nested stringified JSONs, and so we need to recursively parse these.

API 的"奇怪行为"——返回嵌套的 stringified JSON。Claude Code 在这里需要递归 parse（但注释也说 "TODO: This needs patching as recursive fields can still be stringified"——这是个已知但未完全解决的边界情况）。

### text —— 空白响应检测

```ts
case 'text':
  if (contentBlock.text.trim().length === 0) {
    logEvent('tengu_model_whitespace_response', {
      length: contentBlock.text.length,
    })
  }
  // Return the block as-is to preserve exact content for prompt caching.
  return contentBlock
```

模型输出了"全空白文本"——上报 `tengu_model_whitespace_response`，但**不修改内容**。

为什么不修改？注释说："preserve exact content for prompt caching"。

这是 prompt caching 的关键考虑：**任何对内容的修改都会破坏 cache prefix 匹配**。如果 normalize 把空白 trim 掉，下次同样的对话历史就 cache miss——成本和延迟都会爆。所以这里只观测（上报指标），不动数据。

### 其它 block —— 透传或简单 parse

```ts
case 'code_execution_tool_result':
case 'mcp_tool_use':
case 'mcp_tool_result':
case 'container_upload':
  // Beta-specific content blocks - pass through as-is
  return contentBlock
case 'server_tool_use':
  if (typeof contentBlock.input === 'string') {
    return {
      ...contentBlock,
      input: (safeParseJSON(contentBlock.input) ?? {}) as {
        [key: string]: unknown
      },
    }
  }
  return contentBlock
default:
  return contentBlock
```

server-side 工具（web_search / code_execution / mcp 等）的结果由 API 直接返回，不需要 Claude Code 再处理，透传即可。`server_tool_use` 单独处理是因为它的 input 也可能是 string，需要 parse。

## 2.7 流式构造与 normalize 的衔接

把 02 和 03 串起来看完整流程：

```
API SSE 流
   │
   ▼
fetch 层 → 解析 SSE → 产生 StreamEvent 流
   │
   ▼
handleMessageFromStream (本篇主题)
   ├─ 流式 delta → 回调 (onUpdateLength / onStreamingText / ...)
   │  └─ UI 实时显示
   └─ 完整 Message 累积出来后:
      onMessage(message)
         │
         ▼
   ┌─────────────────────────────┐
   │ Message 进入 store          │ ← 内部状态变化
   └────────────┬────────────────┘
                │
                ▼
   ┌─────────────────────────────┐
   │ 准备下一轮 API 请求        │
   │  - 收集 messages[]         │
   │  - normalize 全量历史      │
   └────────────┬────────────────┘
                │
                ▼
   ┌─────────────────────────────┐
   │ normalizeContentFromAPI    │ ← 反向标准化 (assistant 消息内 tool_use 解析等)
   └────────────┬────────────────┘
                │
                ▼
   ┌─────────────────────────────┐
   │ normalizeMessagesForAPI    │ ← [03-08] 主题
   └────────────┬────────────────┘
                │
                ▼
            API 请求
```

注意 `normalizeContentFromAPI` 不在 normalize 主循环 (`normalizeMessagesForAPI`) 里调——它通常在 message 流式累积完成、加入 store 之前调用，对**单条 assistant message 的 content[]** 做标准化。然后这条标准化过的 message 才进入 store，下一轮 normalize 主循环把它和别的 message 一起处理。

## 2.8 几个设计判断回顾

### 1. handleMessageFromStream 自身无状态

设计上让分发器只做"事件 → 回调"路由，所有状态由调用方持有。好处：

- 测试容易（mock 8 个回调即可）；
- 重用容易（不同 UI 框架可以用不同的状态管理）；
- 不会因为状态错乱导致连锁问题。

### 2. partial JSON 累积阶段不 parse

streaming 时不尝试 parse partial JSON——只累加。这避免了：

- 反复 parse 失败的 CPU 开销；
- 误把"中间不合法状态"当成最终结果。

直到 `normalizeContentFromAPI` 阶段才一次性 parse + fallback。

### 3. parse 失败软失败，不阻断对话

JSON parse 失败时回退 `{}` + 上报。这让模型在下一轮看到"工具调用没参数"，自然会重试或换种调用方式。比 throw 异常友好得多。

### 4. signature_delta 排除指标

防止"基础设施 token"污染观测指标。这种细节体现"指标完整性比覆盖率更重要"的工程审美。

### 5. text content 不 trim 以保护 cache

宁可上报指标也不修改内容——prompt caching 的命中率优先于"消息整洁"。

## 2.9 小结

- assistant message 不是构造出来的，是从 SSE 流式事件累积出来的；
- `handleMessageFromStream` 是中央分发器，8 个回调把事件路由到上层；
- 6 类事件 + 4 类 content_block_delta，每种都有明确状态切换语义；
- partial JSON 的特殊处理：streaming 时只累加、最后阶段 parse、失败软回退；
- `normalizeContentFromAPI` 做"API 回流方向"的标准化——和 `normalizeMessagesForAPI` 互补；
- 几个隐性设计判断：分发器无状态、parse 推后、软失败、指标完整性、cache 优先。

下一篇 → [03 normalize 主循环与三路分发](./03-normalize主循环与三路分发.md)
