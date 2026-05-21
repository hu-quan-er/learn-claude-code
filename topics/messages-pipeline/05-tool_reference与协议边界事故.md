# 05 tool_reference 与协议边界事故

> 整个 messages-pipeline 里最有故事性的一篇。一个工业级 LLM 协议陷阱、一次事故 (#21049)、两套互斥的修复方案、一次 5-arm A/B、最后用 feature gate 让两套方案同时共存。

## 5.1 故事开头：functions 标签的双重身份

Anthropic 服务器在编码消息时把不同内容包装成不同标签：

- 系统 tools 段渲染成 `<functions>` 包装的工具列表
- tool_use block 渲染成 `<function_calls>` 包装的调用结构
- tool_result block 渲染成 `<function_results>` 包装的结果结构

后两者在 wire 上有清晰边界——模型识别"哪段是工具调用、哪段是工具结果"。

**问题来自 tool-search beta 的 `tool_reference` block**。tool_reference 是 tool-search 引入的新 content block——模型可以在 tool_result 里嵌入一个引用，指向"我从 tool-search 那里加载了这个工具，这是它的描述"。服务器在渲染时把 tool_reference 展开成同样的 `<functions>` 包装——和系统 tools 段同名。

## 5.2 协议陷阱：functions 结束符 + 一段 sibling 文本 = 学到停止信号

考虑这种 user message 结构：tool_result 内嵌一个 tool_reference，紧跟一个 sibling text block。这条消息在 wire 上的渲染会让 functions 段的关闭标签紧跟着普通文本——而下一条 user message 又以 `\n\nHuman:` 开头。

模型看到的 token 流大致是这样：

工具结果开始 → 工具列表开始 → 工具描述 → 工具列表结束 → 工具结果结束 → 一些普通文本 → 双换行 → "Human:" → 下一段用户消息

注意标记位置：紧贴 functions 段闭合之后，文本块和 Human 前缀挨着。这正好是 **下一个 user turn 的开始标志**。模型多轮看到这种 pattern 之后会学到："functions 闭合标签 + 一段文本 + 双换行 + Human: = 一个 turn 结束的边界"。

学到之后会发生什么？**模型自己也开始生成这个 pattern 作为停止信号**——在某些 tool_result 后面输出一个 functions 关闭标签 + 双换行 + Human:，从而提前结束自己的回复。

这就是 #21049 的核心：模型从协议层的副作用中**学到了错误的停止信号**。

## 5.3 5-arm A/B 与 92%→0%

源码注释（`messages.ts:2604-2609`）：

> any sibling after tool_result renders as `[function_results-close]\n\nHuman:<...>` on the wire. Repeated mid-conversation, this teaches capy to emit `Human:` at a bare tail → 3-token empty end_turn. A/B (sai-20260310-161901) validated: smoosh into tool_result.content → 92% → 0%.

`messages.ts:1909-1928` 的 `relocateToolReferenceSiblings` 注释更明确：

> When a tool_result contains tool_reference, the server expands it to a functions block. Any text siblings appended to that same user message (auto-memory, skill reminders, etc.) create a second human-turn segment right after the functions-close tag — an anomalous pattern the model imprints on. At a later tool-results tail, the model completes the pattern and emits the stop sequence. See #21049 for mechanism and **five-arm dose-response**.

提到 **5-arm dose-response**——这意味着 Anthropic 做了一次 5 个实验组的 A/B：

- 不同剂量的"tool_reference + sibling"出现频率；
- 测量"模型提前停止"的发生率；
- 看是不是真的剂量-效应关系（让事故链路确认）。

92% → 0% 是 A/B 测试编号 `sai-20260310-161901` 的结果——把 sibling smoosh 进 tool_result.content 后，提前停止率从 92% 降到 0%。

## 5.4 两套修复方案

事故确认后，工程团队做了**两套**修复，用 feature gate 切换。

### 方案 A：预防式注入 (`TOOL_REFERENCE_TURN_BOUNDARY`)

源码 `messages.ts:179`：

```ts
const TOOL_REFERENCE_TURN_BOUNDARY = 'Tool loaded.'
```

主循环 user case 里（`messages.ts:2159-2185`）：

```ts
if (
  !checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_toolref_defer_j8m')
) {
  const contentAfterStrip = normalizedMessage.message.content
  if (
    Array.isArray(contentAfterStrip) &&
    !contentAfterStrip.some(
      b =>
        b.type === 'text' &&
        b.text.startsWith(TOOL_REFERENCE_TURN_BOUNDARY),
    ) &&
    contentHasToolReference(contentAfterStrip)
  ) {
    normalizedMessage = {
      ...normalizedMessage,
      message: {
        ...normalizedMessage.message,
        content: [
          ...contentAfterStrip,
          { type: 'text', text: TOOL_REFERENCE_TURN_BOUNDARY },
        ],
      },
    }
  }
}
```

逻辑：

1. 检查这条 user message 是否含 tool_reference；
2. 检查是否已经有 `TOOL_REFERENCE_TURN_BOUNDARY` 起始的 text sibling（幂等保护）；
3. 都不满足时**主动加一个 text sibling**，内容是固定字符串 `'Tool loaded.'`。

**为什么这能修复**？因为模型本来会从 wire 上"functions 闭合 → 普通文本 → 双换行 → Human:" 这种 pattern 学错。注入这个固定 sibling 后，wire 上变成：

工具结果开始 → 工具列表开始 → 工具描述 → 工具列表结束 → 工具结果结束 → "Tool loaded." → 双换行 → "Human:" → 下一段用户消息

这条 "Tool loaded." 看起来像"系统在告知一个事件"，比原本随机的 sibling 文本（来自 auto-memory / skill reminder 等）**更稳定、更像一个合法的 turn 边界**。模型在大量这种 pattern 后学到的是"Tool loaded. + Human: = 正常 turn 边界"，而不是"普通文本 + Human: = 停止信号"。

**这是一个非常巧妙的"用稳定 pattern 替换不稳定 pattern"的策略**——不消灭副作用，而是让副作用以可控形式出现。

注释解释（`messages.ts:2139-2151`）：

> Server renders tool_reference expansion as `<functions>...</fn>` (same tags as the system prompt's tool block). When this is at the prompt tail, capybara models sample the stop sequence at ~10% (A/B: 21/200 vs 0/200 on v3-prod). A sibling text block inserts a clean `\n\nHuman: ...` turn boundary. Injected here (API-prep) rather than stored in the message so it never renders in the REPL, and is auto-skipped when strip* above removes all tool_reference content.

关键细节：

- 注入发生在 **API 准备阶段**，不持久化到 store——所以 REPL 里看不到这条 "Tool loaded."；
- 上文的 strip 如果把所有 tool_reference 都剥掉了，注入条件就不满足——这是个隐性的"strip 与 inject 协同"；
- 注释提到的 ~10% 概率 (21/200 vs 0/200) 是 v3-prod 上的实验数据。

### 方案 B：搬移式修复 (`relocateToolReferenceSiblings`)

`messages.ts:1933-1987`：

```ts
function relocateToolReferenceSiblings(
  messages: (UserMessage | AssistantMessage)[],
): (UserMessage | AssistantMessage)[] {
  const result = [...messages]

  for (let i = 0; i < result.length; i++) {
    const msg = result[i]!
    if (msg.type !== 'user') continue
    const content = msg.message.content
    if (!Array.isArray(content)) continue
    if (!contentHasToolReference(content)) continue

    const textSiblings = content.filter(b => b.type === 'text')
    if (textSiblings.length === 0) continue

    // Find the next user message with tool_result but no tool_reference.
    let targetIdx = -1
    for (let j = i + 1; j < result.length; j++) {
      const cand = result[j]!
      if (cand.type !== 'user') continue
      const cc = cand.message.content
      if (!Array.isArray(cc)) continue
      if (!cc.some(b => b.type === 'tool_result')) continue
      if (contentHasToolReference(cc)) continue
      targetIdx = j
      break
    }

    if (targetIdx === -1) continue // No valid target; leave in place.

    // Strip text from source, append to target.
    result[i] = {
      ...msg,
      message: {
        ...msg.message,
        content: content.filter(b => b.type !== 'text'),
      },
    }
    const target = result[targetIdx] as UserMessage
    result[targetIdx] = {
      ...target,
      message: {
        ...target.message,
        content: [
          ...(target.message.content as ContentBlockParam[]),
          ...textSiblings,
        ],
      },
    }
  }

  return result
}
```

逻辑：

1. 找到所有"含 tool_reference 的 user message"；
2. 提取它们的 text sibling block；
3. 把这些 sibling 搬到**下一条**含 tool_result 但不含 tool_reference 的 user message；
4. 如果找不到目标（已经是末尾），保持原地不动。

**为什么这能修复**？因为模型从 pattern 里学到的是"含 tool_reference 的消息后跟 sibling 文本 + Human:"。搬走 sibling 后，含 tool_reference 的消息就**没有 sibling 了**——pattern 消失，模型学不到。

注释里也说了一个边界情况（`messages.ts:1925-1928`）：

> If no valid target exists (tool_reference message is at/near the tail), siblings stay in place. That's safe: a tail ending in a human turn (with siblings) gets an Assistant: cue before generation; only a tail ending in bare tool output (no siblings) lacks the cue.

末尾时 sibling 保留——因为如果末尾是 tool_reference 消息且后面没有更多 user message，那么这条本来就要发给 API、等模型回复。Anthropic 服务器在生成时会附加 `Assistant:` cue，本来就有 turn 边界——不会触发学习问题。

### 5.4.1 两套方案的本质差异

| 维度 | 方案 A (注入) | 方案 B (搬移) |
|---|---|---|
| 思路 | 用稳定 pattern 替换不稳定 pattern | 消除 pattern 本身 |
| 副作用 | wire 上多一段 "Tool loaded." 文本 | 后续 user message 的 sibling 变多 |
| 幂等性 | 通过 startsWith 判断保证 | 搬完后源没 text sibling，第二遍无效 |
| 失败模式 | "Tool loaded." 自身可能被学成奇怪 pattern | 找不到目标时 sibling 不动 |
| feature gate | `tengu_toolref_defer_j8m` **OFF** 时启用 | `tengu_toolref_defer_j8m` **ON** 时启用 |

## 5.5 为什么保留两套——双 fallback 设计

按理说找到更优方案（B）后，应该把 A 移除。但代码里两套都保留了，用 feature gate 切换。

`messages.ts:2153-2158` 注释：

> Gated OFF when tengu_toolref_defer_j8m is active — that gate enables relocateToolReferenceSiblings in post-processing below, which moves existing siblings to a later non-ref message instead of adding one here. This injection is itself one of the patterns that gets relocated, so skipping it saves a scan. When gate is off, this is the fallback (same as pre-#21049 main).

`messages.ts:2295-2300` 注释：

> Runs after merge (siblings are in place) and before ID tagging (so tags reflect final positions). When gate is OFF, this is a noop and the TOOL_REFERENCE_TURN_BOUNDARY injection above serves as fallback.

设计意图：

- **灰度发布安全**：方案 B 刚上线时，先用 gate 给一部分用户启用、其余仍用方案 A。如果 B 有未发现的问题，立刻切回 A 是单 gate flip 的事；
- **永远有一个生效**：gate ON → 方案 B 修复；gate OFF → 方案 A 修复。**永远不会两个都不生效**——这是个稳定性保险；
- **两个都不互斥**：A 是在 forEach 中注入，B 是 forEach 后的 pass。即使两个都"假启用"，注入会幂等保护，搬移也只搬非 boundary 的 sibling。但**不会两个都启用**——A 是 `if (!gate ON)` 才执行。

这种"两套修复并存 + gate 切换"的设计在工业代码里是**渐进发布的标准做法**：

```
        新方案上线 → 1% 用户 → 监控指标
              ↓ (无回归)
        提到 10% → 监控
              ↓
        提到 50% → 监控
              ↓
        提到 100%
              ↓
        旧方案代码保留 N 周做应急回滚保险
              ↓
        确认稳定 → 移除旧方案
```

注释里"pre-#21049 main"暗示方案 A 是 #21049 修复之前 main 分支上的状态——很可能 A 本身就是更早的某个事故修复。这条防御链路是**多次事故沉淀下来的**。

## 5.6 `contentHasToolReference` —— 边界判定

`messages.ts:1778-1786`：

```ts
function contentHasToolReference(
  content: ContentBlockParam[],
): boolean {
  return content.some(
    block =>
      block.type === 'tool_result' &&
      Array.isArray(block.content) &&
      block.content.some(isToolReferenceBlock),
  )
}
```

两层结构判定：

- 顶层：content 数组里有 `tool_result` block；
- 嵌套：tool_result.content 是数组，且其中含 `tool_reference` block。

任何分支不满足返回 false。这种"必须满足完整结构"的判定避免误判——只对真正的 tool_reference 触发修复。

## 5.7 相关的两个剥离函数

除了修复，还有两个 strip 函数处理 tool_reference 的不同失效情况：

### `stripToolReferenceBlocksFromUserMessage`

`messages.ts:1677-1731`，**tool-search 整体未启用**时调用——剥离所有 tool_reference block（beta-only 不能进普通 API）。如果 tool_result.content 被剥光了，留下占位文本 `'[Tool references removed - tool search not enabled]'`。

### `stripUnavailableToolReferencesFromUserMessage`

`messages.ts:1541-1613`，**tool-search 启用但某些工具消失**时调用——剥离指向已不存在工具的 reference（如 MCP server 断了）。同样如果被剥光留占位 `'[Tool references removed - tools no longer available]'`。

normalize 主循环 user case 里根据 `isToolSearchEnabledOptimistic()` 决定调哪个：

```ts
let normalizedMessage = message
if (!isToolSearchEnabledOptimistic()) {
  normalizedMessage = stripToolReferenceBlocksFromUserMessage(message)
} else {
  normalizedMessage = stripUnavailableToolReferencesFromUserMessage(
    message,
    availableToolNames,
  )
}
```

注意 strip 在 inject 之前：先剥离，再判断有没有 tool_reference 决定要不要 inject。

## 5.8 故事总结：从事故到双 fallback

整理整个故事的时间线（从注释和 gate 名反推）：

```
T0  (远古): tool_use / tool_result 边界设计 → 服务器渲染成 function_calls / function_results 标签
T1  (?):    某次模型学到 "</fn>\n\nHuman:" 是停止信号 → 添加 sibling text 作为 turn boundary 的旧修复
T2  (?):    旧修复运行良好
T3  (?):    tool-search beta 引入 tool_reference content block
T4  (?):    tool_reference 在 tool_result 内被服务器展开成 <functions> 标签
T5  (#21049 事故): 模型在 tool_reference + sibling 的 pattern 上学到错误停止信号 (~10%)
T6  (调查):  5-arm A/B 确认剂量-效应关系
T7  (方案 A): 在 tool_reference 消息上注入 'Tool loaded.' sibling 作为稳定 turn 标记
T8  (修复部分): 92% → 0% 效果显著
T9  (反思):  注入的 sibling 仍然在 tool_reference 消息上 — 不如搬走更彻底
T10 (方案 B): relocateToolReferenceSiblings 实现 — 把 sibling 搬到下一条非 tool_reference user
T11 (灰度):  feature gate `tengu_toolref_defer_j8m` 上线 — gate ON 走 B, OFF 走 A
T12 (维护):  两套并存以备回滚
```

注释里没有给出具体日期，但从 `tengu_toolref_defer_j8m` 这个 gate 名（看起来是某个 issue 编号 j8m）可以推测这是个**有明确 issue/项目编号**的修复 — 工程团队对它的追踪非常细。

## 5.9 几个可以提炼的设计判断

读完这段历史，有几条经验值得记：

### 1. 协议层副作用会泄漏到模型行为里

最反直觉的事实：模型不只是看你给它什么内容，还会看到内容的"形状"。同一个文本放在不同位置，模型学到的东西不同。这是 LLM 系统设计里独有的问题——不能只考虑"内容对不对"，还要考虑"内容出现的 pattern 对不对"。

### 2. 用稳定 pattern 替换不稳定 pattern

方案 A 不试图消灭"functions 关闭 + sibling text + Human" 这种序列——而是让 sibling 变成一个固定字符串 "Tool loaded."，让 pattern 稳定到模型能学正确含义。这种思路在很多场景都好用：与其消除噪声，不如把噪声标准化。

### 3. A/B 实验的剂量-效应分析

5-arm A/B 不是简单"有没有效果"——而是不同剂量下的响应曲线。这种实验设计能让你**确认因果**而不只是相关。事故调查的金标准。

### 4. 修复方案保留旧路径作为 fallback

新方案上线不等于旧方案删除。两套共存 + feature gate 切换是工业代码标准做法。等新方案稳定 N 周（甚至更长）才考虑清理旧路径。

### 5. 注释里写下事故编号和 A/B 编号

代码注释引用 `#21049`、`sai-20260310-161901` 这种编号——让未来的工程师能查到完整背景。注释是协作的最低成本，但能省下未来几小时的考古工作。

### 6. 用 startsWith 做幂等保护

方案 A 的注入用 `text.startsWith(TOOL_REFERENCE_TURN_BOUNDARY)` 判断已存在——这样 normalize 可以反复跑而不重复注入。这也意味着 `TOOL_REFERENCE_TURN_BOUNDARY` 字符串必须**前缀稳定**（不会被后续逻辑改前缀）——这又约束了 `joinTextAtSeam` 把 `\n` 加在 a 的末尾而不是 b 的开头（[04](./04-合并smoosh与hoist三连击.md#4.4.2)）。

整个设计是相互纠缠的。

## 5.10 小结

- `<functions>` 标签的"双重身份"（系统 tools 段 + tool_reference 展开）导致模型从 pattern 中学到错误停止信号；
- #21049 事故，5-arm A/B 确认 ~10% 影响，方案 A (注入 'Tool loaded.') 把率降到 0%；
- 后续方案 B (relocateToolReferenceSiblings) 更彻底——搬移而不是注入；
- 双 fallback 设计：feature gate `tengu_toolref_defer_j8m` 切换，gate ON 用 B、OFF 用 A，永远有一个生效；
- 注释里写下事故和 A/B 编号——这是工业代码协作的标配；
- 设计相互纠缠：startsWith 幂等保护 → 约束 boundary 字符串前缀稳定 → 约束 joinTextAtSeam 的 `\n` 位置。

下一篇 → [06 协议合规过滤层](./06-协议合规过滤层.md)
