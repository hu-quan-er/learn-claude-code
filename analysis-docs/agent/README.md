# AgentTool 子代理专题

Claude Code 2.1.88 中 `Agent` 工具的完整实现：模型如何 spawn 子代理、有几种派发模式、并行/串行如何调度、模型怎么选、内部 `runAgent` 怎么 fork 父对话状态、子代理工具集怎么过滤、worktree 和 fork 各自隔离了什么、agent memory 怎么管理、退出后如何清场。

## 阅读顺序

| 顺序 | 文档 | 适合解决的问题 |
|------|------|----------------|
| 1 | [00-总览与源码地图](./00-总览与源码地图.md) | 整体由哪些文件组成、关键概念分别指什么 |
| 2 | [01-Agent定义与可派发类型](./01-Agent定义与可派发类型.md) | 能派发哪些类型？built-in / 用户级 / 团队级各自怎么定义 |
| 3 | [02-派发模式](./02-派发模式.md) | sync / async / teammate / remote / fork 五种 spawn 模式各自的行为 |
| 4 | [03-并行与串行调度](./03-并行与串行调度.md) | 多个子代理是并行还是串行？模型怎么决定并行？ |
| 5 | [04-模型选择优先级](./04-模型选择优先级.md) | 子代理用什么模型？五级优先级 + Bedrock 区域继承 |
| 6 | [05-AgentTool工具入口](./05-AgentTool工具入口.md) | input schema、permission、UI 串流、5 种 status 分支 |
| 7 | [06-runAgent核心机制](./06-runAgent核心机制.md) | fork 时哪些状态被共享 / 克隆 / 隔离，内部 query 循环 |
| 8 | [07-工具继承与隔离机制](./07-工具继承与隔离机制.md) | 子代理可见 tools 数组的计算，Fork + Worktree 两种隔离 |
| 9 | [08-AgentMemory与Snapshot](./08-AgentMemory与Snapshot.md) | user / project / local 三层 scope + snapshot 同步 |
| 10 | [09-生命周期与端到端模拟](./09-生命周期与端到端模拟.md) | cleanup + resume + 一个完整数据流模拟 |

## 最短结论

子代理本质上是**用一次工具调用换一次完整对话的工作量**：模型把"我接下来要花 10 个 turn 做的事"打包成一个 prompt 发给 `Agent` 工具，runtime fork 出新 query 循环，子代理用私有 context 干完，结果通过 `tool_result` 回填父对话。

```text
父代理 assistant tool_use (Agent)
  → AgentTool.call() 分流到 5 种派发模式
  → runAgent() fork：共享 AppState/mcp/hooks，克隆 readFileState，隔离 agentId/abort/transcript
  → 子代理内部 query 循环
  → 结果（completed / async_launched / teammate_spawned / remote_launched）
  → mapToolResultToToolResultBlockParam 转 tool_result
  → 父代理下一轮继续决策
```

5 种派发模式（在 `outputSchema` / `InternalOutput` 上能看到 status 枚举）：

| 模式 | 触发条件 | 行为 |
|------|----------|------|
| 同步 | 默认 | 父 turn 内 `await` 等子代理结果 |
| `async_launched` | `run_in_background: true` | 立即返回 agentId，父代理继续，完成时通过通知队列回 |
| `teammate_spawned` | `team_name` + `name`（swarm 模式） | 在独立 tmux pane 启动 teammate，mailbox 通信 |
| `remote_launched` | `isolation: "remote"`（ant 内部） | 提交到 CCR 云端，跨进程，必定后台 |
| fork（隐式 status: completed） | `subagent_type` 省略且 `isForkSubagentEnabled()` | 父 messages 整体继承到子，prompt cache 共享 |
