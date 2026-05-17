# 01 - V1 TodoWrite 实现

## 数据模型

**文件**：`src/utils/todo/types.ts`

```typescript
type TodoStatus = 'pending' | 'in_progress' | 'completed'

type TodoItem = {
  content: string
  status: TodoStatus
  activeForm: string
}

type TodoList = TodoItem[]
```

三个字段的含义：

| 字段 | 说明 |
|------|------|
| `content` | 命令式任务描述，例如 `Run tests` |
| `activeForm` | 进行时描述，例如 `Running tests`，用于模型和 UI 表达“正在做什么” |
| `status` | `pending` / `in_progress` / `completed` 三态 |

V1 没有任务 ID、依赖关系、owner、更新时间等字段。它的设计假设是“模型每次提交完整列表”，而不是对单个 item 做增量 patch。

## 工具注册与启用

**文件**：`src/tools.ts`

`getAllBaseTools()` 总是把 `TodoWriteTool` 放进基础工具集合：

```typescript
TodoWriteTool
```

但真正可用性由 `TodoWriteTool.isEnabled()` 决定：

```typescript
isEnabled() {
  return !isTodoV2Enabled()
}
```

也就是说，V2 开启时，`TodoWrite` 不再作为当前任务系统暴露给模型；工具集合会改为额外加入：

```typescript
TaskCreateTool
TaskGetTool
TaskUpdateTool
TaskListTool
```

系统提示中的任务管理指令也会从 `[TaskCreate, TodoWrite]` 中选择当前启用的工具名。这样模型在不同运行模式下会被引导使用对应的任务系统。

## 原生工具定义

**文件**：`src/tools/TodoWriteTool/TodoWriteTool.ts`

```typescript
name = 'TodoWrite'
searchHint = 'manage the session task checklist'
strict = true
isEnabled = () => !isTodoV2Enabled()
userFacingName = () => ''
shouldDefer = true
maxResultSizeChars = 100_000
```

输入：

```typescript
type TodoWriteInput = {
  todos: Array<{
    content: string
    status: 'pending' | 'in_progress' | 'completed'
    activeForm: string
  }>
}
```

输出：

```typescript
type TodoWriteOutput = {
  oldTodos: TodoList
  newTodos: TodoList
  verificationNudgeNeeded?: boolean
}
```

权限与分类：

```typescript
checkPermissions(input) {
  return { behavior: 'allow', updatedInput: input }
}

renderToolUseMessage() {
  return null
}

toAutoClassifierInput(input) {
  return `${input.todos.length} items`
}
```

含义：

- Todo 更新不需要用户批准。
- 工具调用不直接渲染成用户可见消息。
- 主要效果是更新内部状态，并给模型返回 `tool_result`。

## 执行流程

```text
Assistant tool_use: TodoWrite({ todos })
  → ToolExecutor 调用 TodoWriteTool.call()
  → 取 todoKey = context.agentId ?? getSessionId()
  → oldTodos = appState.todos[todoKey] ?? []
  → 如果 todos 全部 completed，则状态中保存 []
  → 否则状态中保存传入 todos
  → 返回 oldTodos / newTodos / verificationNudgeNeeded
  → mapToolResultToToolResultBlockParam() 生成 tool_result 文本
```

关键实现：

```typescript
const todoKey = context.agentId ?? getSessionId()
const oldTodos = appState.todos[todoKey] ?? []
const allDone = todos.every(_ => _.status === 'completed')
const newTodos = allDone ? [] : todos

context.setAppState(prev => ({
  ...prev,
  todos: {
    ...prev.todos,
    [todoKey]: newTodos,
  },
}))
```

这里有一个细节：当所有任务完成时，写入 `AppState.todos` 的是空数组，用于清理当前列表；但工具输出里的 `newTodos` 返回的是传入的 `todos`。所以“模型可见的工具结果”和“AppState 实际保存值”在全完成场景下不完全一致。

返回给模型的 `tool_result` 固定以这句话开头：

```text
Todos have been modified successfully. Ensure that you continue to use the todo list to track your progress. Please proceed with the current tasks if applicable
```

如果触发 verification nudge，会追加一段要求调用 verification agent 的提醒。注意它的 `userFacingName()` 是空字符串，因此用户界面不会把它当成一个显眼的工具名展示。

## AppState 存储

**文件**：`src/state/AppStateStore.ts`

```typescript
todos: { [agentId: string]: TodoList }
```

初始状态为：

```typescript
todos: {}
```

key 的选择规则：

| 场景 | key |
|------|-----|
| 主会话 | `getSessionId()` |
| 子代理 | `context.agentId` |

这种结构让主会话和多个子代理的 todo list 可以隔离。子代理结束时，`runAgent.ts` 会删除对应 key，避免大量后台 agent 留下空列表造成状态泄漏。

## Transcript 恢复

**文件**：`src/utils/sessionRestore.ts`

V1 没有文件持久化。SDK / 非交互式 resume 时，恢复逻辑从历史消息倒序扫描最后一次 `TodoWrite` `tool_use`：

```text
messages 从后往前扫描
  → 找 assistant message
  → 找 tool_use.name === 'TodoWrite'
  → 读取 input.todos
  → 用 TodoListSchema 校验
  → 写回 AppState.todos[getSessionId()]
```

源码注释也明确说明：交互式模式使用 V2 文件态 tasks，因此 `AppState.todos` 在 interactive 中不承担恢复职责。

全完成场景有一个边界：

| 位置 | 最终值 |
|------|--------|
| `AppState.todos["sess-001"]` | `[]` |
| tool result `newTodos` | 全部 completed 的原始列表 |
| transcript 中的 `tool_use.input.todos` | 全部 completed 的原始列表 |

如果之后从 transcript resume，`sessionRestore.ts` 读取的是最后一次 `TodoWrite.input.todos`，而不是读取运行时曾经清空后的 `AppState`。所以一个“全完成”的列表在 resume 时可能重新变成 completed 列表。

## Reminder 注入

**文件**：`src/utils/attachments.ts`

```typescript
export const TODO_REMINDER_CONFIG = {
  TURNS_SINCE_WRITE: 10,
  TURNS_BETWEEN_REMINDERS: 10,
}
```

`getTodoReminderTurnCounts()` 从消息尾部倒序扫描：

- 跳过 thinking message。
- 找最近一次 assistant `TodoWrite` `tool_use`。
- 找最近一次 `todo_reminder` attachment。
- 计算距离现在经过了多少个 assistant turn。

满足两个条件才会注入 reminder：

```text
turnsSinceLastTodoWrite >= 10
turnsSinceLastReminder >= 10
```

`getTodoReminderAttachments()` 会提前跳过以下情况：

| 条件 | 原因 |
|------|------|
| 当前工具列表里没有 `TodoWrite` | 模型不可调用，不提醒 |
| 当前工具列表里有 `SendUserMessage` / brief 通道 | brief 是主通信机制，TodoWrite 只是 side channel，提醒会干扰流程 |
| 没有历史 messages | 无法计算轮次 |

当触发 reminder 时，attachment 内容为：

```typescript
{
  type: 'todo_reminder',
  content: todos,
  itemCount: todos.length,
}
```

**文件**：`src/utils/messages.ts`

`todo_reminder` 会被转换为 meta user message，并包进 system reminder：

```text
The TodoWrite tool hasn't been used recently...
Here are the existing contents of your todo list:
1. [status] content
```

这条消息明确要求模型不要向用户提及 reminder。也就是说，reminder 是“对模型的轻推”，不是用户界面提示。

## 远程任务进度

**文件**：`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`

远程 agent 的 SDK log 中也可能包含 `TodoWrite`。本地轮询远程任务时，会在 log 增长后重新扫描最后一次 TodoWrite：

```typescript
todoList: logGrew
  ? extractTodoListFromLog(accumulatedLog)
  : prevTask.todoList
```

`RemoteSessionProgress` 只展示完成数：

```text
completed / total
```

所以远程任务里的 TodoList 主要是“背景任务进度摘要”，不是本地主交互列表。

## V1 要点

```text
模型负责生成完整 TodoList
  → TodoWriteTool 只做 schema 校验和状态落盘到 AppState
  → 后一次 TodoWrite 覆盖前一次列表
  → 全 completed 时 AppState 被清空
  → resume 依赖 transcript 中最后一次 tool_use input
```
