# messages-pipeline 消息规范化管线专题

> 基于 `claude-code@2.1.88` 还原源码（`claude-code-sourcemap/restored-src/`）。
> 行号对应当前快照；版本升级会漂移。

## 阅读顺序

| # | 文档 | 主题 | 你会学到 |
|---|---|---|---|
| 00 | [总览与全管线](./00-总览与全管线.md) | 入口 | normalize 在 query loop 中的位置、5512 行 messages.ts 的分区地图、全管线 ASCII 图 |
| 01 | [消息类型与构造](./01-消息类型与构造.md) | 基础 | 4 类 Message 的字段差异、`createX` 族函数、`SYNTHETIC_*` 系列、`INTERRUPT_MESSAGE` 与 reject 措辞设计 |
| 02 | [流式事件与 assistant 增量构造](./02-流式事件与assistant增量构造.md) | 输入侧 | `handleMessageFromStream` 状态机、`content_block_*` 事件、tool_use input JSON 边解析边累积、whitespace / parse-fail 上报点 |
| 03 | [normalize 主循环与三路分发](./03-normalize主循环与三路分发.md) | 核心 | `normalizeMessagesForAPI` 1989-2370 逐段、user / assistant / attachment 三路 case、PDF/image error 的反向追溯剥离 |
| 04 | [合并 smoosh 与 hoist 三连击](./04-合并smoosh与hoist三连击.md) | 内容重塑 | `mergeAdjacent → mergeUserMessages → joinTextAtSeam → mergeUserContentBlocks → smoosh → hoist` 的多层嵌套，每一层为什么必要 |
| 05 | [tool_reference 与协议边界事故](./05-tool_reference与协议边界事故.md) | 协议陷阱 | `#21049` 事故复盘、`TOOL_REFERENCE_TURN_BOUNDARY` 注入、`relocateToolReferenceSiblings`、双 fallback 灰度策略 |
| 06 | [协议合规过滤层](./06-协议合规过滤层.md) | 协议合规 | filterOrphanedThinking / filterTrailingThinking / filterWhitespace / ensureNonEmpty / sanitizeError / validateImages 各自对应的具体 400 |
| 07 | [attachment normalizer](./07-attachment-normalizer.md) | 友方注入 | `normalizeAttachmentForAPI` 25+ case 系统盘点、与 [prompt-injection 04] 的预算章对照看 |
| 08 | [端到端模拟与 feature gate 时间线](./08-端到端模拟与feature-gate时间线.md) | 收尾 | 一次完整 normalize 推演 + `tengu_*` gate 演化故事（从注释里反推） |

## 一句话总结这个专题

`utils/messages.ts` 是 Claude Code 在用户输入、模型流式输出、工具结果、attachment 这四类原始消息和 Anthropic Messages API **协议合规消息**之间的**那一层**。它 5512 行，是整个项目里最长的单文件之一。代码的每一段几乎都有 A/B 实验编号、事故链接、92%→0% 的修复痕迹——这一层是 Claude Code **被生产环境打磨**最重的部分。

读完这个专题，你会理解：

- 为什么 LLM agent 不能"把原始消息直接发给 API"——协议、训练、性能、缓存四方面都有约束；
- "smoosh / hoist / merge" 这些怪名字到底在解决什么；
- `<system-reminder>` 标签机制（已在 [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md) 讲过）在这个管线里出现的位置；
- `#21049` 事故如何引出 `relocateToolReferenceSiblings` + `TOOL_REFERENCE_TURN_BOUNDARY` 的"双 fallback 设计"；
- 流式协议（SSE + content_block_delta）和静态消息（已落盘的 Message）之间的边界怎么处理；
- 友方注入（attachment 25+ 种 case）如何被统一收编进 user message。

## 关键源码地图

| 区段 | 行 | 角色 |
|---|---|---|
| 消息常量与构造族 | `messages.ts:200-630` | `INTERRUPT_MESSAGE`/`SYNTHETIC_*`/`createUserMessage`/`createAssistantMessage`/`createToolResultStopMessage` |
| 索引与查询（UI 用） | `messages.ts:700-1470` | `normalizeMessages` / `buildMessageLookups` / `getToolResultIDs` / `getSiblingToolUseIDs` |
| 工具引用辅助 | `messages.ts:1481-1790` | `reorderAttachmentsForAPI` / `stripToolReferenceBlocks` / `appendMessageTagToUserMessage` |
| 清洗与重塑（pre-normalize） | `messages.ts:1797-1988` | `ensureSystemReminderWrap` / `smooshSystemReminderSiblings` / `sanitizeErrorToolResultContent` / `relocateToolReferenceSiblings` |
| **`normalizeMessagesForAPI` 主入口** | `messages.ts:1989-2370` | normalize 的指挥中枢，含三路分发 |
| 合并族 | `messages.ts:2372-2516` | `merge*` 家族 / `hoistToolResults` / `joinTextAtSeam` |
| `smooshIntoToolResult` 底层 | `messages.ts:2534-2598` | 折叠 SR-text 进 tool_result 的具体实现 |
| `normalizeContentFromAPI` | `messages.ts:2651-2751` | API 回流消息的反向标准化（tool_use input JSON 解析、whitespace 检测） |
| `handleMessageFromStream` | `messages.ts:2930-3094` | SSE 流式事件分发（4 类事件 × 5 类 content_block） |
| 标签包装 | `messages.ts:3097-3134` | `wrapInSystemReminder` / `wrapMessagesInSystemReminder`（与 prompt-injection 专题共用） |
| `normalizeAttachmentForAPI` | `messages.ts:3453-4600+` | 25+ 种 attachment 类型 → user message 的转换大 switch |
| 协议合规过滤族 | `messages.ts:4800-5050` | `filterOrphanedThinking` / `filterTrailingThinking` / `filterWhitespace` / `ensureNonEmpty` |
| tool-result 配对修复 | `messages.ts:5371-5450` | `ensureToolResultPairing` / `SYNTHETIC_TOOL_RESULT_PLACEHOLDER` |

## 与其它专题的关系

- **`<system-reminder>` 机制**：本专题第 03/04 篇会再触及，但深度展开在 [prompt-injection 01](../prompt-injection/01-system-reminder标签机制.md)；
- **smoosh 反 prompt-injection 的语义解耦**：本专题讲"为什么 smoosh"和"smoosh 怎么实现"，[prompt-injection 01] 讲"smoosh 在防御里扮演什么角色"——互补关系；
- **attachment 类型**：本专题第 07 篇盘点 case，对照 [prompt-injection 04 友方注入预算](../prompt-injection/04-attachments与surfacer预算.md)看"配额从哪里施加"；
- **agent 子代理**：[agent 06 runAgent](../agent/06-runAgent核心机制.md) 调 `normalizeMessagesForAPI`，本专题讲它内部怎么工作；
- **TodoList tools 拼接**：[todolist 06](../todolist/06-tools数组中todo工具的prompt拼接与内容.md) 讲 tools[] 字段拼接，本专题讲 messages[] 字段构造——一起构成了一次 API 请求的全部 payload。

下一篇 → [00 总览与全管线](./00-总览与全管线.md)
