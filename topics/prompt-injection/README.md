# prompt-injection 防御机制专题

> 基于 `claude-code@2.1.88` 还原源码（`claude-code-sourcemap/restored-src/`）。
> 行号对应当前快照；版本升级会漂移。

## 阅读顺序

| # | 文档 | 主题 | 你会学到 |
|---|---|---|---|
| 00 | [威胁模型与防御层次](./00-威胁模型与防御层次.md) | 总览 | 攻击源 × 防御层次映射、全文件地图、防御边界 |
| 01 | [system-reminder 标签机制](./01-system-reminder标签机制.md) | 核心抽象 | `<system-reminder>` 的语义、`smooshSystemReminderSiblings` 合并、与 user/tool_result 边界关系 |
| 02 | [FileRead 双护栏](./02-FileRead双护栏.md) | 内容护栏 | 行号前缀 `N\t` 防注入、读后 malware system-reminder、FileEdit 如何借用前缀做锚定 |
| 03 | [Unicode 清洗管线](./03-Unicode清洗管线.md) | 字符级防御 | `partiallySanitizeUnicode` 91 行逐行、迭代不动点、Hidden Unicode 攻击防护表 |
| 04 | [attachments 与 surfacer 预算](./04-attachments与surfacer预算.md) | 友方注入约束 | `MAX_MEMORY_BYTES` / 5×4KB 单 turn 上限 / 60KB session 上限、轮次间隔配额 |
| 05 | [外部工具内容隔离](./05-外部工具内容隔离.md) | 防御广度 | Bash/Web/MCP 工具结果如何被标注、什么时候靠 tag、什么时候靠 system prompt 兜底 |
| 06 | [元防御指令与端到端模拟](./06-元防御指令与端到端模拟.md) | 收尾 | `prompts.ts:191` 两条元指令逐字解读、一次"恶意 README 文件读"的端到端推演 |

## 一句话总结整套设计

Claude Code 防 prompt-injection 不是单点护栏，而是 **6 层叠加** 的纵深防御：

1. **字符层**：`recursivelySanitizeUnicode` 把 Unicode 隐写攻击（零宽字符 / 双向控制符 / Tag chars）在入口拦截。
2. **格式层**：`addLineNumbers` 给所有文件内容打上 `N\t` 前缀，让模型可以无歧义地区分"文件内容 vs 包装文本"——文件里再写 "ignore previous instructions" 也只是 `42\tignore previous instructions`。
3. **标签层**：`<system-reminder>` 给"系统注入的内容"一个明确身份标签，且通过 `smooshSystemReminderSiblings` 强制把它折叠进 tool_result 内部，避免在线协议层泄露出可被模型误学的 stop pattern。
4. **预算层**：`MAX_MEMORY_BYTES = 4096` × 5/turn × 60KB/session 三级配额，把"友方注入"（CLAUDE.md、相关记忆、计划提醒）本身也当成需要约束的资源。
5. **指令层**：system prompt 显式告诉模型 ① `<system-reminder>` 是脱钩元信息 ② 工具结果可能含恶意内容、可疑时 flag 给用户。
6. **行为层**：FileRead 后强制注入 `CYBER_RISK_MITIGATION_REMINDER`——"可以分析恶意代码、但不许 improve/augment"，把"模型被骗"的最终行为面也封闭。

每一层都不假设上一层完美。这是这套设计最值得学习的地方。

## 关键源码地图

| 模块 | 文件 | 行 | 角色 |
|---|---|---|---|
| 系统提示元指令 | `src/constants/prompts.ts` | 190-191 | 元防御指令（标签元定义 + flag-to-user） |
| `<system-reminder>` 包装 | `src/utils/messages.ts` | 3097-3134 | `wrapInSystemReminder` / `wrapMessagesInSystemReminder` |
| smoosh 合并管线 | `src/utils/messages.ts` | 1789-1873 / 2534-2598 | `ensureSystemReminderWrap` / `smooshSystemReminderSiblings` / `smooshIntoToolResult` |
| Unicode 清洗 | `src/utils/sanitization.ts` | 1-91 | `partiallySanitizeUnicode` / `recursivelySanitizeUnicode` |
| 行号前缀 | `src/utils/file.ts` | 287-328 | `addLineNumbers` / `stripLineNumberPrefix` |
| FileRead malware reminder | `src/tools/FileReadTool/FileReadTool.ts` | 695-738 | `CYBER_RISK_MITIGATION_REMINDER` 注入 |
| 友方注入预算 | `src/utils/attachments.ts` | 255-292 | `RELEVANT_MEMORIES_CONFIG` / `MAX_MEMORY_BYTES` |
| 外部内容入口 | 各 Tool 的 `mapToolResultToToolResultBlockParam` | — | tool_result 隐式分隔 |

更细的对应表见 [00-威胁模型与防御层次](./00-威胁模型与防御层次.md)。
