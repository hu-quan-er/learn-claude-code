# 02 - V2 Task 工具实现

## 数据模型

**文件**：`src/utils/tasks.ts`

```typescript
type Task = {
  id: string
  subject: string
  description: string
  activeForm?: string
  owner?: string
  status: 'pending' | 'in_progress' | 'completed'
  blocks: string[]
  blockedBy: string[]
  metadata?: Record<string, unknown>
}
```

相比 V1，V2 增加了：

- 稳定 ID。
- 负责人 `owner`。
- 阻塞关系 `blocks` / `blockedBy`。
- 任意元数据 `metadata`。
- 文件持久化和跨进程协同。

## 文件存储与并发控制

V2 任务存储路径：

```text
{CLAUDE_CONFIG_HOME}/tasks/{sanitize(taskListId)}/{taskId}.json
```

任务列表 ID 优先级：

```text
1. CLAUDE_CODE_TASK_LIST_ID
2. in-process teammate 的 teamName
3. CLAUDE_CODE_TEAM_NAME / getTeamName()
4. leaderTeamName
5. getSessionId()
```

创建任务时使用 lockfile 串行化：

```text
ensureTaskListLockFile()
  → lockfile.lock()
  → findHighestTaskId()
  → writeFile("{id}.json")
  → notifyTasksUpdated()
```

`HIGH_WATER_MARK_FILE = '.highwatermark'` 用于记录曾经分配过的最大 ID，避免 reset 后复用旧 ID。

## 工具总览

这里的“原生”指 `src/tools.ts#getAllBaseTools()` 里随 Claude Code 基础工具集注册的任务工具，不包括 MCP、插件或用户自定义工具。

| 工具 | 所属版本 | 启用条件 | 读写性质 | 主要用途 |
|------|----------|----------|----------|----------|
| `TodoWrite` | V1 | `!isTodoV2Enabled()` | 写 `AppState.todos` | 非交互式 / SDK 场景中一次性提交完整 checklist |
| `TaskCreate` | V2 | `isTodoV2Enabled()` | 写文件 | 创建一个新的 pending task |
| `TaskUpdate` | V2 | `isTodoV2Enabled()` | 写文件 | 更新任务字段、状态、依赖、owner，或删除任务 |
| `TaskGet` | V2 | `isTodoV2Enabled()` | 只读 | 按 ID 读取单个 task |
| `TaskList` | V2 | `isTodoV2Enabled()` | 只读 | 列出当前 task list 的所有非内部 task |

共同点：

- 都通过 `buildTool({...})` 定义，满足 `ToolDef<InputSchema, Output>`。
- `shouldDefer: true`，在工具搜索启用时可能先只以名字出现在 `<available-deferred-tools>` 中。
- `renderToolUseMessage() { return null }`，调用本身不直接渲染为普通用户消息。
- `maxResultSizeChars: 100_000`，避免任务列表或结果无限膨胀。

## TaskCreate

**文件**：`src/tools/TaskCreateTool/TaskCreateTool.ts`

```typescript
name = 'TaskCreate'
searchHint = 'create a task in the task list'
isEnabled = () => isTodoV2Enabled()
isConcurrencySafe = () => true
userFacingName = () => 'TaskCreate'
shouldDefer = true
```

输入：

```typescript
type TaskCreateInput = {
  subject: string
  description: string
  activeForm?: string
  metadata?: Record<string, unknown>
}
```

输出：

```typescript
type TaskCreateOutput = {
  task: {
    id: string
    subject: string
  }
}
```

主要副作用：

```text
createTask(getTaskListId(), {
  subject,
  description,
  activeForm,
  status: 'pending',
  owner: undefined,
  blocks: [],
  blockedBy: [],
  metadata,
})
  → 写 ~/.claude/tasks/{taskListId}/{id}.json
  → notifyTasksUpdated()
  → executeTaskCreatedHooks(...)
  → 如果 hook 有 blockingError，deleteTask(taskId) 并抛错
  → setAppState(expandedView = 'tasks')
```

返回给模型：

```text
Task #<id> created successfully: <subject>
```

所以 `TaskCreate` 只负责创建 pending 任务；它不会自动把新任务标成 `in_progress`。开始执行时需要模型再调用 `TaskUpdate`。

## TaskUpdate

**文件**：`src/tools/TaskUpdateTool/TaskUpdateTool.ts`

```typescript
name = 'TaskUpdate'
searchHint = 'update a task'
isEnabled = () => isTodoV2Enabled()
isConcurrencySafe = () => true
userFacingName = () => 'TaskUpdate'
shouldDefer = true
```

输入：

```typescript
type TaskUpdateInput = {
  taskId: string
  subject?: string
  description?: string
  activeForm?: string
  status?: 'pending' | 'in_progress' | 'completed' | 'deleted'
  addBlocks?: string[]
  addBlockedBy?: string[]
  owner?: string
  metadata?: Record<string, unknown | null>
}
```

输出：

```typescript
type TaskUpdateOutput = {
  success: boolean
  taskId: string
  updatedFields: string[]
  error?: string
  statusChange?: {
    from: string
    to: string
  }
  verificationNudgeNeeded?: boolean
}
```

主要副作用：

```text
getTask(taskListId, taskId)
  → 不存在：返回 success=false，不抛 fatal error
  → status='deleted'：deleteTask(taskListId, taskId)
  → status='completed'：先 executeTaskCompletedHooks(...)
  → subject/description/activeForm/status/owner/metadata：合并更新到任务文件
  → addBlocks/addBlockedBy：调用 blockTask() 建立双向依赖
  → owner 变化且 swarm 开启：writeToMailbox(owner, task_assignment)
  → setAppState(expandedView = 'tasks')
```

metadata 的特殊语义：

```text
metadata[key] = value   → 设置或覆盖
metadata[key] = null    → 删除该 metadata key
```

返回给模型：

```text
Updated task #<taskId> <updatedFields...>
```

如果任务不存在，返回：

```text
Task not found
```

这里特意把“任务不存在”做成普通 `tool_result`，而不是工具错误。原因是任务可能已经被清理或被别的 agent 更新，模型可以自行恢复。

`TaskUpdate` 的几个关键行为：

- 设置 `completed` 前执行 TaskCompleted hooks，blocking error 会阻止完成。
- swarm 模式下，teammate 将任务置为 `in_progress` 且未指定 owner 时，会自动填入当前 agent name。
- owner 变化时，会向新 owner 的 mailbox 写 `task_assignment`。
- `addBlocks` / `addBlockedBy` 最终都通过 `blockTask()` 建立双向依赖。
- 主线程关闭 3 个以上任务且无验证任务时，同样触发 verification nudge。

## TaskGet

**文件**：`src/tools/TaskGetTool/TaskGetTool.ts`

```typescript
name = 'TaskGet'
searchHint = 'retrieve a task by ID'
isEnabled = () => isTodoV2Enabled()
isConcurrencySafe = () => true
isReadOnly = () => true
userFacingName = () => 'TaskGet'
shouldDefer = true
```

输入：

```typescript
type TaskGetInput = {
  taskId: string
}
```

输出：

```typescript
type TaskGetOutput = {
  task: null | {
    id: string
    subject: string
    description: string
    status: 'pending' | 'in_progress' | 'completed'
    blocks: string[]
    blockedBy: string[]
  }
}
```

主要行为：

```text
getTask(getTaskListId(), taskId)
  → 找不到：{ task: null }
  → 找到：返回任务核心字段
```

返回给模型的文本：

```text
Task #<id>: <subject>
Status: <status>
Description: <description>
Blocked by: #...
Blocks: #...
```

如果找不到：

```text
Task not found
```

## TaskList

**文件**：`src/tools/TaskListTool/TaskListTool.ts`

```typescript
name = 'TaskList'
searchHint = 'list all tasks'
isEnabled = () => isTodoV2Enabled()
isConcurrencySafe = () => true
isReadOnly = () => true
userFacingName = () => 'TaskList'
shouldDefer = true
```

输入：

```typescript
type TaskListInput = {}
```

输出：

```typescript
type TaskListOutput = {
  tasks: Array<{
    id: string
    subject: string
    status: 'pending' | 'in_progress' | 'completed'
    owner?: string
    blockedBy: string[]
  }>
}
```

主要行为：

```text
listTasks(getTaskListId())
  → 过滤 metadata._internal 为 true 的内部任务
  → 找出 completed task IDs
  → 每个任务的 blockedBy 过滤掉已经 completed 的 blocker
  → 返回扁平任务列表
```

返回给模型的文本：

```text
#1 [in_progress] Implement login form UI
#2 [pending] Add client-side validation
#4 [pending] Run tests [blocked by #3]
```

如果没有任务：

```text
No tasks found
```

这个工具是 teammate 完成任务后寻找下一项工作的关键入口：`TaskUpdate` 在 teammate 完成任务时会提示“Call TaskList now to find your next available task”。

## UI 刷新

**文件**：`src/hooks/useTasksV2.ts`

V2 UI 使用一个单例 `TasksV2Store`：

```text
useSyncExternalStore()
  → 订阅 singleton store
  → store 监听 onTasksUpdated()
  → fs.watch(tasksDir)
  → 50ms debounce fetch
  → fallback 5s poll
```

它解决两个问题：

1. 多个组件同时需要任务列表时，不重复创建多个 `fs.watch`。
2. 同进程工具调用可通过 `notifyTasksUpdated()` 立即刷新，跨进程变更则依赖文件 watcher 和 fallback poll。

完成后的隐藏逻辑：

```text
如果所有任务 completed
  → 5 秒后再次确认仍全部 completed
  → resetTaskList(taskListId)
  → UI hidden = true
```

## UI 渲染

**文件**：`src/components/TaskListV2.tsx`

渲染策略：

- completed：tick 图标、删除线。
- in_progress：实心方块、加粗。
- pending：空方块。
- blocked：变暗并显示 `blocked by #id`。
- teammate owner 活跃时显示 `@owner` 和最近活动。
- 终端高度不足时最多显示 10 条，并优先保留：
  1. 最近 30 秒完成的任务。
  2. 进行中任务。
  3. 未阻塞的 pending。
  4. 较早完成的任务。

`ctrl+t` 绑定在 `app:toggleTodos`，如果存在 teammate 视图，则循环：

```text
none → tasks → teammates → none
```

否则在：

```text
none ↔ tasks
```

之间切换。`expandedView` 会持久化回 `showExpandedTodos` / `showSpinnerTree`，保持向后兼容。
