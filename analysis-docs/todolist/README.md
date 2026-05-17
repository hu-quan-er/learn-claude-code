# TodoList 专题

这个目录拆分说明 Claude Code 2.1.88 中 todolist 的完整实现链路：它如何作为工具定义暴露给模型、模型如何返回 `tool_use`、runtime 如何执行工具并持久化状态，以及 V1 `TodoWrite` 和 V2 `Task*` 两套实现的差异。

## 阅读顺序

| 顺序 | 文档 | 适合解决的问题 |
|------|------|----------------|
| 1 | [00-总览与源码地图](./00-总览与源码地图.md) | 先建立 V1/V2 的整体心智模型 |
| 2 | [01-V1-TodoWrite实现](./01-V1-TodoWrite实现.md) | 理解非交互式 / SDK 场景的 checklist 如何保存和恢复 |
| 3 | [02-V2-Task工具实现](./02-V2-Task工具实现.md) | 理解交互式 CLI 的文件态任务系统和原生 todo 工具定义 |
| 4 | [03-工具传递与tool_use决策链路](./03-工具传递与tool_use决策链路.md) | 理解工具 schema 如何传给模型、模型如何决策并返回 `tool_use` |
| 5 | [04-端到端数据流模拟](./04-端到端数据流模拟.md) | 通过模拟数据流串起创建、更新、完成、清理 |
| 6 | [05-设计取舍与阅读路径](./05-设计取舍与阅读路径.md) | 快速复盘设计取舍、脏点和源码阅读路径 |

## 最短结论

TodoList 的“产生者”不是 Claude Code runtime 自动从用户文本里解析出来的任务列表，而是模型主动返回的工具调用：

```text
系统提示 + tools schema
  → 模型决定调用 TodoWrite / TaskCreate / TaskUpdate
  → assistant message 中出现 tool_use block
  → Claude Code 校验 input_schema 并执行本地工具
  → 工具写 AppState 或任务文件
  → runtime 把执行结果作为 tool_result 发回模型
```

V1 和 V2 的区别在状态模型：

| 问题 | V1 TodoWrite | V2 Task tools |
|------|--------------|---------------|
| Todo 从哪来 | 模型一次提交完整 `todos` 数组 | 模型调用 `TaskCreate` 逐个创建 |
| 如何更新 | 再次提交完整数组，覆盖旧数组 | `TaskUpdate(taskId, fields)` 增量更新文件 |
| 状态主存储 | `AppState.todos[todoKey]` | `~/.claude/tasks/{taskListId}/{id}.json` |
| 是否有 ID | 没有 | 有 |
| 是否能表达依赖 | 不能 | `blocks` / `blockedBy` |
| 是否适合多 agent | 较弱，按 agentId 隔离 | 更强，可共享 taskListId、owner、mailbox |
