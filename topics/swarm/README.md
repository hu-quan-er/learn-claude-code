# Multi-agent Swarm 专题

> 基于 `claude-code@2.1.88` 还原源码（`claude-code-sourcemap/restored-src/`）。
> 行号对应当前快照；版本升级会漂移。

## 阅读顺序

| # | 文档 | 主题 | 你会学到 |
|---|---|---|---|
| 00 | [总览与代码地图](./00-总览与代码地图.md) | 总览 | 10K 行分区、3 backend 对比、激活路径、与 [agent 专题](../agent/) 的本质区别 |
| 01 | [激活机制与团队配置](./01-激活机制与团队配置.md) | 入口 | `isAgentSwarmsEnabled` 双重 gate、`TEAM_LEAD_NAME` 常量、`~/.claude/teams/{name}/config.json` 结构、TeamCreate / TeamDelete 工具 |
| 02 | [Teammate 身份与上下文](./02-Teammate身份与上下文.md) | 身份 | AsyncLocalStorage in-process / CLI flag tmux / dynamicTeamContext 优先级、`isTeamLead` / `getAgentName` |
| 03 | [Pane backend (tmux + iTerm2)](./03-Pane-backend-tmux-iterm2.md) | backend ① | PaneBackend 接口、TmuxBackend 764 行、ITermBackend + it2Setup RPC、detection + registry |
| 04 | [In-process backend](./04-In-process-backend.md) | backend ② | InProcessBackend 339 行：无真正 pane 时如何 stub 接口、layout 元数据的纯内存实现 |
| 05 | [消息系统与 SendMessage](./05-消息系统与SendMessage.md) | 通信 | Mailbox 通用类、file-based teammateMailbox 1183 行、inbox 结构、lockfile 并发、`<task-notification>` 自动送达 |
| 06 | [协议消息状态机](./06-协议消息状态机.md) | 协议 | shutdown_request/_response、plan_approval_request/_response、`request_id` echo、JSON 嵌套在文本、`collapseTeammateShutdowns` 折叠机制 |
| 07 | [In-process Runner](./07-In-process-Runner.md) | 运行时 | `inProcessRunner.ts` 1552 行：同进程跑 teammate、context 隔离、流式回到 lead、生命周期与 abort |
| 08 | [Permission Sync](./08-Permission-Sync.md) | 权限同步 | `permissionSync.ts` 928 行：worker 没本地权限决策、`leaderPermissionBridge` 把请求传回 lead、reconnection 策略 |
| 09 | [spawnMultiAgent 批量派发](./09-spawnMultiAgent批量派发.md) | 批量 | `spawnMultiAgent.ts` 1093 行：一次派多个 agent 的 layout、顺序、部分失败 cleanup |
| 10 | [Team Memory 与持久化](./10-Team-Memory与持久化.md) | 共享记忆 | teamMem 跨 teammate 共享、`memdir/teamMem*`、`teamMemSecretGuard` 防 secret 泄漏 |
| 11 | [端到端模拟与设计哲学](./11-端到端模拟与设计哲学.md) | 收尾 | "team-lead 派 3 个 teammates 协作完成 task" 完整推演 + 与 [agent] / [messages-pipeline] 串联 + 设计哲学 |

## 一句话总结这个专题

Multi-agent Swarm 是 Claude Code 在 **单个 Agent 之上加一层"团队协调"** 的系统——多个 Claude 实例（teammate）通过文件 mailbox 异步通信、被一个 leader 协调、共享一份 task list 和 memory。

它比 [AgentTool 专题](../agent/) 讲的"sub-agent 派发"更上一层：sub-agent 是父 agent 通过工具调用同步等待子 agent 返回结果的 RPC 模式；swarm 是多个**对等 actor**长期并发运行、通过 mailbox 交换消息的 actor 模式。**两者的设计哲学截然不同**——前者像函数调用，后者像 Erlang。

读完这个专题，你会理解：

- **3 个 backend 怎么共存**——tmux（用户在终端里看 split pane）、iTerm2（用户在 iTerm2 里看 split pane）、in-process（没真 pane，teammate 跑在同进程的 AsyncLocalStorage 里）；
- 为什么用 **文件 mailbox + lockfile**（而不是内存队列 / unix socket / 网络）——支持跨进程跨终端、不需要服务进程、写入立即持久化便于 resume；
- **`isTeamLead` 单例角色**是怎么贯穿整个系统的——permission 决策由 lead 弹窗、worker 通过 bridge 把请求传回 lead；
- **协议消息**（shutdown_request/plan_approval_request）怎么用嵌套 JSON 在自由文本字段里塞一个状态机；
- 为什么 `inProcessRunner.ts` 有 1552 行——AsyncLocalStorage 隔离一份完整的 Claude 实例需要处理 cwd / agentId / teamContext / mailbox / permissionSync / abort / 退出清理 等 7+ 个独立状态；
- `spawnMultiAgent.ts` 1093 行做的事——一次性派多个 agent 时怎么按顺序起 pane、失败时 cleanup 已起的、让 lead 知道哪些起好了。

## 关键源码地图（10K 行）

| 模块 | 文件 | 行数 |
|------|------|----:|
| **门控** | `utils/agentSwarmsEnabled.ts` | 44 |
| **常量** | `utils/swarm/constants.ts` | 33 |
| **团队 config** | `tools/TeamCreateTool/TeamCreateTool.ts` + `prompt.ts` | 354 |
| | `tools/TeamDeleteTool/TeamDeleteTool.ts` + `prompt.ts` | 156 |
| | `utils/swarm/teamHelpers.ts` | 683 |
| | `utils/teamDiscovery.ts` + `teamMemoryOps.ts` | 169 |
| **身份与上下文** | `utils/teammate.ts` + `teammateContext.ts` + `inProcessTeammateHelpers.ts` | 490 |
| **Pane backend** | `backends/types.ts` + `registry.ts` + `detection.ts` + `PaneBackendExecutor.ts` | 1257 |
| | `backends/TmuxBackend.ts` | 764 |
| | `backends/ITermBackend.ts` + `it2Setup.ts` | 615 |
| | `backends/teammateModeSnapshot.ts` | 87 |
| **In-process backend** | `backends/InProcessBackend.ts` | 339 |
| **消息系统** | `tools/SendMessageTool/SendMessageTool.ts` + `prompt.ts` | 966 |
| | `utils/teammateMailbox.ts` | 1183 |
| | `utils/mailbox.ts` | 73 |
| | `utils/collapseTeammateShutdowns.ts` | 55 |
| **In-process Runner** | `utils/swarm/inProcessRunner.ts` | **1552** |
| | `utils/swarm/spawnInProcess.ts` + `spawnUtils.ts` | 474 |
| | `utils/swarm/teammateInit.ts` + `teammateLayoutManager.ts` | 236 |
| **Permission Sync** | `utils/swarm/permissionSync.ts` | 928 |
| | `utils/swarm/leaderPermissionBridge.ts` + `reconnection.ts` | 173 |
| **批量派发** | `tools/shared/spawnMultiAgent.ts` | 1093 |
| **Team Memory** | `memdir/teamMemPaths.ts` + `teamMemPrompts.ts` | 392 |
| | `services/teamMemorySync/teamMemSecretGuard.ts` 等 | ~200 |
| **Task** | `tasks/InProcessTeammateTask/InProcessTeammateTask.tsx` | 125 |
| **UI / hooks** | components/teams/、Spinner、messages、tasks 周边 | ~500 |
| **合计** | | **~10000** |

## 与其它专题的关系

- **[agent 专题](../agent/)**：那里讲 AgentTool 的 **sub-agent 派发**——父 agent 通过工具调用同步等待子 agent 完成。Swarm 在它之上：多个**并发**Claude 实例长期协作，通过消息通信。子集关系：swarm 用 spawnMultiAgent 派 teammate，teammate 自己也能调 AgentTool 派 sub-agent。
- **[messages-pipeline 专题](../messages-pipeline/)**：messages-pipeline 07 提到的 `teammate_mailbox` / `team_context` attachment 类型——本专题展开它们如何被生成、写到磁盘、被 normalize 转回 user message。
- **[prompt-injection 专题](../prompt-injection/)**：teammate 之间的消息是新的 prompt-injection 入口——本专题第 05/06 篇会触及"如何防 teammate 给 lead 发恶意消息"。
- **[bashtool 专题](../bashtool/)**：worker teammate 跑 Bash 时不能本地决策权限——通过 [08 Permission Sync](./08-Permission-Sync.md) 把请求传回 lead 弹窗。

下一篇 → [00 总览与代码地图](./00-总览与代码地图.md)
