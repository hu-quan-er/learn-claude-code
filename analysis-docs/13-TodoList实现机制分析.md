# 13 - TodoList 实现机制分析

本文是 TodoList 专题的入口页。完整拆分文档放在 [`todolist/`](./todolist/) 目录下，避免单篇文档继续膨胀。

---

## 核心结论

Claude Code 2.1.88 里的 todolist 不是单一实现，而是两条并存的任务跟踪通道：

| 通道 | 主要入口 | 默认使用场景 | 存储位置 | UI 展示 |
|------|----------|--------------|----------|---------|
| TodoWrite / V1 | `TodoWrite` | 非交互式会话、SDK、远程任务日志 | `AppState.todos` 内存态，恢复时从 transcript 扫描 | 本地主流程基本不直接展示；远程任务用进度计数 |
| Task tools / V2 | `TaskCreate` / `TaskUpdate` / `TaskGet` / `TaskList` | 交互式 CLI 默认启用 | `~/.claude/tasks/{taskListId}/*.json` 文件态 | `TaskListV2` 持久展示，`ctrl+t` 可展开 |

源码里看到的 `TodoList` 类型只是 V1 的列表结构；交互式 CLI 中用户感知到的“任务列表”主要由 V2 `Task` 文件系统驱动。两者通过 `isTodoV2Enabled()` 分流：

```typescript
export function isTodoV2Enabled(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_TASKS)) {
    return true
  }
  return !getIsNonInteractiveSession()
}
```

因此：

- 交互式 REPL：默认使用 V2 `Task*` 工具。
- 非交互式 / SDK：默认使用 V1 `TodoWrite`。
- 设置 `CLAUDE_CODE_ENABLE_TASKS=1`：即使非交互式也强制使用 V2。

---

## 拆分文档

| 文档 | 内容 |
|------|------|
| [00-总览与源码地图](./todolist/00-总览与源码地图.md) | V1/V2 分流、源码入口、阅读顺序 |
| [01-V1-TodoWrite实现](./todolist/01-V1-TodoWrite实现.md) | V1 数据模型、工具生命周期、AppState、transcript 恢复、reminder |
| [02-V2-Task工具实现](./todolist/02-V2-Task工具实现.md) | V2 Task 数据模型、文件存储、五个原生 todo 工具定义、UI 刷新 |
| [03-工具传递与tool_use决策链路](./todolist/03-工具传递与tool_use决策链路.md) | 工具如何转成 API `tools` 数组、模型如何返回 `tool_use`、runtime 如何回填 `tool_result` |
| [04-端到端数据流模拟](./todolist/04-端到端数据流模拟.md) | 模拟 V1 和 V2 从用户请求到任务创建、更新、完成、清理的完整数据流 |
| [05-设计取舍与阅读路径](./todolist/05-设计取舍与阅读路径.md) | V1/V2 的设计优缺点、脏点、建议源码阅读路径 |

一句话总结：**V1 TodoWrite 是模型上下文里的轻量 checklist；V2 Task tools 是交互式 CLI 的可持久化任务系统。**
