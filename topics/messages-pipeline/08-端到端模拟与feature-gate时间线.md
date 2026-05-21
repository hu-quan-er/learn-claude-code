# 08 端到端模拟与 feature gate 时间线

> 收尾篇。先用一次完整的对话轮次把前 7 篇的所有机制串成一条数据流，再从源码注释里反推这套管线的演化时间线。

## 8.1 端到端模拟：一次完整轮次

设定一个具体场景：

- 用户已经和 Claude Code 对话了几轮；
- 当前 store 里有一段历史；
- 用户刚输入新的一句话："继续，看看 config.ts 里有没有问题"；
- auto-memory surfacer 在后台检索到一份相关记忆；
- 上一轮模型调用了 FileRead 工具，结果还在历史里。

我们追踪这一轮 API 请求的完整 normalize 过程。

### T0：store 里的原始 message 数组

```
[0] user        : "帮我看下这个项目"            (origin=人类键盘)
[1] assistant   : [text "好的，我先看下结构"]
[2] assistant   : [tool_use Bash "ls"]          (message.id = msg_A)
[3] user        : [tool_result "src/ config.ts ..."]
[4] assistant   : [text "项目结构清楚了"]        (message.id = msg_B)
[5] progress    : (Bash 执行中的进度上报)
[6] user        : "继续，看看 config.ts 里有没有问题"  (origin=人类键盘)
[7] attachment  : { type: 'relevant_memories', memories: [...] }
```

注意几个"杂质"：
- `[5] progress` —— UI 进度消息，不该进 API；
- `[7] attachment` —— 友方注入，需要展开。

### T1：进入 `normalizeMessagesForAPI`

#### 段 1 预处理

**reorderAttachmentsForAPI**：attachment `[7]` 试图上浮。它紧贴 `[6] user`，已经在合理位置——不动。

**filter virtual**：没有 isVirtual 消息，不变。

**build stripTargets**：扫描有没有 `isSyntheticApiErrorMessage`——没有。stripTargets 为空。

#### 段 2 主 forEach

**filter**：`[5] progress` 被过滤掉。剩下 7 条进 forEach。

```
[0] user        → push. result = [user0]
[1] assistant   → 同 id 查找无果 → push. result = [user0, asst1]
[2] assistant   → tool_use 标准化:
                  - 找到 Bash 工具, normalizeToolInputForAPI
                  - tool-search 未启用 → 显式构造只含标准字段的 tool_use block
                  → 同 id (msg_A) 查找: asst1 的 id 不是 msg_A → push
                  result = [user0, asst1, asst2]
[3] user        → strip tool_reference (无) / strip 错误源 (无)
                  → 注入 TOOL_REFERENCE_TURN_BOUNDARY? contentHasToolReference=false → 不注入
                  → last 是 asst2 (assistant), 不 merge → push
                  result = [user0, asst1, asst2, user3]
[4] assistant   → tool_use 标准化 (无 tool_use, 跳过)
                  → 同 id (msg_B) 查找: 往前扫 user3(tool_result, 继续) asst2(id=msg_A≠msg_B, continue)
                    asst1(id≠msg_B, continue) user0(不是 assistant 不是 tool_result → break)
                  → 没找到同 id → push
                  result = [..., user3, asst4]
[6] user        → strip / 注入检查都不触发
                  → last 是 asst4 (assistant), 不 merge → push
                  result = [..., asst4, user6]
[7] attachment  → normalizeAttachmentForAPI({type:'relevant_memories', ...})
                  → 模式 B: 每份记忆一条 isMeta user message
                  → ensureSystemReminderWrap (gate ON): 包 <system-reminder>
                  → last 是 user6 (user!) → reduce mergeUserMessagesAndToolResults
                    → user6 内容 [text("继续...")] + memory user [SR-text]
                    → mergeUserContentBlocks: user6 末尾是 text 不是 tool_result → 直接 concat
                  result = [..., asst4, user6+memory]
```

forEach 后 result：

```
[user0]  "帮我看下这个项目"
[asst1]  [text "好的..."]
[asst2]  [tool_use Bash ls]
[user3]  [tool_result "src/ config.ts..."]
[asst4]  [text "项目结构清楚了"]
[user6]  [text "继续...", text "<system-reminder>relevant memory</system-reminder>"]
```

#### 段 3 后处理串联

```
1. relocateToolReferenceSiblings (gate ON):
   扫描含 tool_reference 的 user — 无 — noop

2. filterOrphanedThinkingOnlyMessages:
   没有 thinking-only 消息 — noop

3. filterTrailingThinkingFromLastAssistant:
   最后一条是 user6, 不是 assistant — noop

4. filterWhitespaceOnlyAssistantMessages:
   没有全空白 assistant — noop

5. ensureNonEmptyAssistantContent:
   没有空 content[] assistant — noop

6. mergeAdjacentUserMessages + smooshSystemReminderSiblings:
   - mergeAdjacent: 没有相邻 user (user3/user6 被 assistant 隔开) — noop
   - smooshSystemReminderSiblings:
       user6 content = [text("继续..."), text("<system-reminder>...")]
       有 tool_result 吗? — 没有! user6 里只有 text
       → 不满足 smoosh 条件 (smoosh 只折进 tool_result)
       → user6 保持 [text, SR-text] 两个 sibling

7. sanitizeErrorToolResultContent:
   user3 的 tool_result is_error=false — noop

8. appendMessageTagToUserMessage (HISTORY_SNIP):
   给 user0 / user6 加 [id:xxx] (user3 是 tool_result-only, 但它不是 isMeta...
   实际上 user3 也会被加 tag, 除非它 isMeta)

9. validateImagesForAPI:
   没有 image — noop
```

### T2：最终发给 API 的 messages

```
[user]  "帮我看下这个项目\n[id:a3f2x1]"
[asst]  [text "好的，我先看下结构"]
[asst]  [tool_use Bash {command:"ls"}]
[user]  [tool_result "src/ config.ts..."]
[asst]  [text "项目结构清楚了"]
[user]  [text "继续，看看 config.ts 里有没有问题\n[id:b7k9m2]",
         text "<system-reminder>\nrelevant memory: ...\n</system-reminder>"]
```

注意最终结果：
- `progress` 消失了；
- `attachment` 被吸收进 user6；
- 相关记忆以 `<system-reminder>` 形式贴在 user6 末尾；
- 每条非 meta user 带了 `[id:xxx]` 标签。

### T3：一个值得注意的细节

上面 step 6 里，`relevant memory` 的 SR-text **没有被 smoosh**——因为 user6 里没有 tool_result，smoosh 只折叠进 tool_result。

那它会以 sibling text 形式发给 API。这没问题——sibling text 只有跟在 **tool_result 后面**才有协议陷阱（[05](./05-tool_reference与协议边界事故.md) 讲的 `Human:` 模式）。user6 末尾是普通 text sibling，不触发陷阱。

如果换个场景——假设 user6 本身是个 tool_result 消息（比如这一轮模型先调了工具），那 SR-text 就会被 smoosh 进 tool_result.content。**smoosh 是否触发，取决于上下文里有没有 tool_result**——这是 [04](./04-合并smoosh与hoist三连击.md) 讲的逻辑。

### T4：带 tool_result 的对比场景

把场景改一下：假设当前轮次模型先调了 FileRead，attachment 紧跟在 tool_result 之后。那么 forEach 里 attachment case 的合并会不同：

```
last 是 user (含 tool_result)
→ mergeUserMessagesAndToolResults
  → mergeUserContentBlocks
    → a 末尾是 tool_result!
    → universal smoosh (gate ON):
      把 memory SR-text smoosh 进 tool_result.content
→ 结果: tool_result 内部包着 <system-reminder> memory
```

这时模型看到的是：FileRead 的结果里**附带**了一段 `<system-reminder>` 记忆。SR-text 在 tool_result 内部，wire 上没有 sibling，没有 `Human:` 陷阱。

这两个场景对比说明：**同一个 attachment，落到不同的上下文位置，会走不同的合并/smoosh 路径**。normalize 的复杂性正来自"要适配所有上下文位置"。

## 8.2 从注释反推 feature gate 时间线

messages.ts 的注释里散落着大量 feature gate 名、事故编号、A/B 实验编号。把它们拼起来能还原一条演化时间线。

### 涉及的 feature gate

| gate | 控制什么 | 注释证据 |
|---|---|---|
| `tengu_chair_sermon` | universal smoosh + ensureSystemReminderWrap | `messages.ts:2274,2335,2615,5371` |
| `tengu_toolref_defer_j8m` | relocateToolReferenceSiblings vs TOOL_REFERENCE_TURN_BOUNDARY 注入 | `messages.ts:2153,2161,2301` |
| `HISTORY_SNIP` | [id:xxx] 标签 + isMeta 合并规则 | `messages.ts:2351,2414,4149` |
| `EXPERIMENTAL_SKILL_SEARCH` | skill_discovery attachment | `messages.ts:3506` |
| `tengu_amber_prism` | memory correction hint | `messages.ts:188` |
| `CONNECTOR_TEXT` | connector_text content block | `messages.ts:3005,5076` |

### 涉及的事故 / A/B 编号

| 编号 | 含义 | 出处 |
|---|---|---|
| `#21049` | tool_reference + sibling 导致模型学到停止信号 | `messages.ts:2158,2297,1917` |
| `sai-20260310-161901` | smoosh 把 SR-text 折进 tool_result：92%→0% | `messages.ts:2608,1829` |
| `21/200 vs 0/200 on v3-prod` | tool_reference 末尾停止序列采样率 ~10% | `messages.ts:2141-2142` |
| 两个 Slack thread 链接 | smoosh 设计讨论 | `messages.ts:2604-2605` |
| `#21049 five-arm dose-response` | 5 组剂量-效应 A/B | `messages.ts:1917-1918` |

### 反推的演化时间线

注释没有日期，但从 gate 名、事故编号、A/B 编号、"pre-#21049 main"这类措辞，可以反推一条大致顺序：

```
┌─ 阶段 0: 基础管线 ─────────────────────────────────────┐
│  normalizeMessagesForAPI 基本形态                       │
│  - merge 相邻 user (Bedrock 限制)                       │
│  - hoist tool_result                                    │
│  - 基础 filter (whitespace / empty)                     │
└─────────────────────────────────────────────────────────┘
                          │
┌─ 阶段 1: tool_result sibling 的早期修复 ───────────────┐
│  发现 sibling 跟在 tool_result 后会让模型学 "Human:"    │
│  早期用"加 sibling text 作 turn boundary"修复            │
│  (注释 "pre-#21049 main" 指这个状态)                     │
└─────────────────────────────────────────────────────────┘
                          │
┌─ 阶段 2: tool-search beta 引入 tool_reference ─────────┐
│  EXPERIMENTAL_SKILL_SEARCH / tool-search 上线           │
│  tool_reference content block 出现                       │
│  服务器把它展开成 <functions> 标签                       │
└─────────────────────────────────────────────────────────┘
                          │
┌─ 阶段 3: #21049 事故 ──────────────────────────────────┐
│  tool_reference + sibling 让模型学错停止信号 (~10%)     │
│  5-arm dose-response A/B 确认因果                        │
└─────────────────────────────────────────────────────────┘
                          │
┌─ 阶段 4: smoosh 方案 (tengu_chair_sermon) ─────────────┐
│  A/B sai-20260310-161901: smoosh SR-text 进 tool_result │
│  提前停止率 92% → 0%                                     │
│  universal smoosh + ensureSystemReminderWrap 上线        │
└─────────────────────────────────────────────────────────┘
                          │
┌─ 阶段 5: relocate 方案 (tengu_toolref_defer_j8m) ──────┐
│  比"注入 sibling"更彻底——直接搬走 sibling                │
│  双 fallback: gate ON 用 relocate, OFF 用注入            │
└─────────────────────────────────────────────────────────┘
                          │
┌─ 阶段 6: HISTORY_SNIP 新压缩策略 ──────────────────────┐
│  [id:xxx] 标签注入                                       │
│  mergeUserMessages 增加 isMeta 合并规则                  │
└─────────────────────────────────────────────────────────┘
```

注意这个时间线是**反推**——源码注释没给日期，gate 名里的 `j8m`、A/B 编号 `sai-20260310-161901` 里的 `20260310` 看起来像日期（2026-03-10），可作为锚点。但严格说这是基于注释线索的合理推测，标注"反推"。

### 时间线告诉我们什么

1. **管线不是设计出来的，是长出来的**：每个 feature gate、每个 filter pass 都对应一次具体的事故或 A/B。没有人坐下来一次性设计这 5512 行——它是一层层补丁累积的结果。

2. **每个补丁都留下了痕迹**：gate 名、事故编号、A/B 编号、Slack 链接。这些不是噪声——它们让未来的工程师（和 LLM agent）能查到完整背景。

3. **修复会迭代**：同一个问题（sibling-after-tool_result）有过至少 3 代修复：早期 turn-boundary sibling → smoosh → relocate。每一代比上一代更彻底。

4. **旧修复不轻易删除**：阶段 4 的 smoosh 和阶段 5 的 relocate 共存，阶段 1 的注入也作为 fallback 保留。这是渐进发布的纪律。

## 8.3 整个管线的设计哲学总结

读完 8 篇，可以提炼这条管线体现的设计哲学：

### 1. 内部表示与协议表示分离

Claude Code 内部的 `Message` 联合类型有 7 种、字段繁多（uuid / isMeta / isVirtual / origin / toolUseResult ...）。API 协议只认 user/assistant 两种 role。

normalize 这一层就是**翻译层**——让内部表示尽可能丰富（服务于 UI / transcript / resume / rewind），让协议表示尽可能合规（服务于 API）。两者解耦，各自演化。

### 2. 协议陷阱用结构变换解决

API 的隐性约束（tool_result 必须在前、相邻 text 拼接、is_error 只能 text、sibling-after-tool_result 的 `Human:` 陷阱）——全部用纯数据结构变换解决：hoist、joinTextAtSeam、sanitize、smoosh。不依赖运行时检查、不靠"小心翼翼地构造"。

### 3. 错误处理是数据流的分支

`validateImagesForAPI` 抛错 → 上层合成 syntheticError → 进消息历史 → 下轮 normalize 走 stripTargets 反向追溯剥离。错误不是"异常路径"，而是数据流的一个分支，最终汇回正常流程。

### 4. 多 pass 串联，每 pass 单一职责

normalize 后处理段是 9 个 pass 串联。每个 pass 做一件事、单独可测、单独上报指标。代价是 pass 顺序敏感（作者承认脆弱）。这是"可维护性 vs 简洁性"的权衡——选了可维护性。

### 5. feature gate 让灰度发布无缝

`tengu_chair_sermon` / `tengu_toolref_defer_j8m` / `HISTORY_SNIP` 在管线里散布多处决策点。同一管线、不同 gate 路径——新策略可以 1%→10%→100% 灰度，出问题单 flip 回滚。

### 6. 幂等性让管线可反复跑

几乎所有变换都幂等：`ensureSystemReminderWrap` 用 startsWith 判断、`smoosh` 在已 smoosh 的输入上 no-op、`relocate` 搬完后源没 sibling、`appendMessageTag` 不重复加。这让 normalize 可以串联、可以反复跑而不破坏结果。

### 7. 注释承载历史

gate 名、事故编号、A/B 编号、Slack 链接、"建议未来合并"的自批评——注释把"为什么这样设计"完整保留。这是 5512 行能被后人（和 LLM）理解的关键。

### 8. prompt cache 是隐形约束

多处设计为了 cache 命中让步：`relevant_memories` 用存储 header 不重算、`text` content 不 trim 空白、normalize 输出不持久化（uuid 每次变所以不能存）。cache 命中率直接关系成本和延迟——它是个隐形但强力的约束。

## 8.4 这个专题与其它专题的位置

```
                  一次 Claude Code API 请求的完整 payload
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
    ┌──────────────┐    ┌──────────────────┐   ┌──────────────┐
    │ system prompt│    │  messages[]      │   │  tools[]     │
    │              │    │                  │   │              │
    │ context-     │    │ ★ messages-      │   │ todolist 06  │
    │ management   │    │   pipeline       │   │ (tools 拼接) │
    │ 专题         │    │   本专题         │   │              │
    └──────────────┘    └──────────────────┘   └──────────────┘
```

- **本专题**：messages[] 字段怎么从内部 Message 构造出来；
- **context-management 专题**：system prompt 怎么构建、上下文怎么压缩；
- **todolist 06**：tools[] 字段里工具描述怎么拼接；
- 三者合起来 = 一次 API 请求的完整 payload 的全部来源。

另外：
- [agent 06 runAgent](../agent/06-runAgent核心机制.md) 里子代理调 `normalizeMessagesForAPI`——本专题解释它内部；
- [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md) 讲 `<system-reminder>` 在防御里的角色——本专题讲它在管线里的位置。

## 8.5 小结

- 端到端模拟展示：progress 被过滤、attachment 被吸收、SR-text 是否 smoosh 取决于上下文有没有 tool_result；
- 同一 attachment 落在不同位置走不同合并路径——normalize 的复杂性来自"适配所有位置"；
- 从注释反推的演化时间线：基础管线 → 早期 sibling 修复 → tool_reference 引入 → #21049 事故 → smoosh 方案 → relocate 方案 → HISTORY_SNIP；
- 管线不是设计的、是长出来的——每个 gate / filter 对应一次具体事故或 A/B；
- 8 条设计哲学：内部/协议表示分离、协议陷阱用结构解决、错误是数据流分支、多 pass 单一职责、feature gate 灰度、幂等可反复跑、注释承载历史、prompt cache 隐形约束。

---

## 专题结束 · 回到入口

← [README](./README.md) · [00 总览](./00-总览与全管线.md) · [01 消息类型](./01-消息类型与构造.md) · [02 流式事件](./02-流式事件与assistant增量构造.md) · [03 normalize 主循环](./03-normalize主循环与三路分发.md) · [04 合并 smoosh hoist](./04-合并smoosh与hoist三连击.md) · [05 tool_reference 事故](./05-tool_reference与协议边界事故.md) · [06 过滤层](./06-协议合规过滤层.md) · [07 attachment normalizer](./07-attachment-normalizer.md)
