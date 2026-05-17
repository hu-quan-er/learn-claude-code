# 06 - tools 数组中 todo 工具的 prompt 拼接与内容

03 篇讲了管线（`Tool → toolToAPISchema → tools[]`），但没展开**最终进入 API `description` 字段的那一大段自然语言到底是什么**。这一篇把这块补齐：

- 每个 todo 工具的 `description` 字段实际由哪段源码、哪个函数生成；
- 逐字给出最终拼接结果；
- 哪些分支会让同一个工具在不同会话里产出不同 `description`（swarm、feature flag）；
- 一个工具一旦序列化后会被 session 级缓存，mid-session 翻转 flag 不会重拼。

## 关键事实先列在前面

1. API 请求体 `tools[i].description` 字段 = `await tool.prompt(...)` 的字符串，**不是** `tool.description()`。
2. 每个 todo 工具 `tool.prompt()` 的来源：
   - `TodoWriteTool` → 常量 `PROMPT`（模块加载时已经把 `${FILE_EDIT_TOOL_NAME}` 替换成 `"Edit"`）。
   - `TaskCreateTool` → 函数 `getPrompt()`，按 `isAgentSwarmsEnabled()` 拼两种文本。
   - `TaskUpdateTool` → 常量 `PROMPT`。
   - `TaskGetTool` → 常量 `PROMPT`。
   - `TaskListTool` → 函数 `getPrompt()`，按 `isAgentSwarmsEnabled()` 拼两种文本。
3. 模块顶层的 `DESCRIPTION` 常量（比如 `'Update a task in the task list'`）是 `tool.description()` 的返回值，用于 ToolSearch 列表、UI 标题等内部场景，**不会出现在发给模型的 `tools[]` 数组里**。

## 拼接管线复述

`src/utils/api.ts:119` 的 `toolToAPISchema`：

```typescript
base = {
  name: tool.name,
  description: await tool.prompt({
    getToolPermissionContext,
    tools,
    agents,
    allowedAgentTypes,
  }),
  input_schema,
}
```

随后可能附加：

| 字段 | 触发条件 |
|------|----------|
| `strict: true` | `tengu_tool_pear` 开启、`tool.strict === true`、model 支持 structured outputs |
| `eager_input_streaming: true` | 第一方 API + `tengu_fgts` 或 `CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING` |
| `defer_loading: true` | ToolSearch 启用且此工具被识别为 deferred |
| `cache_control: { type: 'ephemeral', ... }` | 调用方传入 cacheControl |

todo 工具里只有 `TodoWriteTool` 声明了 `strict: true`，其它 4 个没有。5 个都声明了 `shouldDefer: true`，因此在 ToolSearch 启用时都可能携带 `defer_loading: true`。

`SWARM_FIELDS_BY_TOOL` 只过滤 `ExitPlanModeV2` 和 `Agent` 工具的 `input_schema` 字段，**不会删 `TaskUpdate` 的 `owner`**。Swarm 分支只通过 `getPrompt()` 影响 `description` 文本。

## Session-stable 缓存（重要）

`toolToAPISchema` 在 `getToolSchemaCache()` 里用 `cacheKey = tool.name`（普通工具）做 key，把 `{ name, description, input_schema, strict, eager_input_streaming }` 一并缓存。注释原文：

> Session-stable base schema ... computed once per session and cached to prevent mid-session GrowthBook flips (tengu_tool_pear, tengu_fgts) or tool.prompt() drift from churning the serialized tool array bytes.

直接后果：

- 一个 session 内 `TaskCreate` 的 `description` 在第一次序列化时就被固化。如果首次序列化时 `isAgentSwarmsEnabled() === false`，本 session 后续即使翻成 true，模型看到的 `TaskCreate.description` 仍是无 teammate 版本。
- `defer_loading` 和 `cache_control` 是请求级的，每次都重新写在 `schema` 上，不进缓存。

## 5 个工具最终 `description` 的实际内容

下面给出的就是 `await tool.prompt(...)` 返回的逐字字符串。空白和缩进按源码模板字面量保留。

### TodoWrite

来源：`src/tools/TodoWriteTool/prompt.ts` 的 `PROMPT` 常量。模板字面量中 `${FILE_EDIT_TOOL_NAME}` 在模块加载时已替换为字符串 `Edit`（见 `src/tools/FileEditTool/constants.ts:2`）。

```text
Use this tool to create and manage a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.
It also helps the user understand the progress of the task and overall progress of their requests.

## When to Use This Tool
Use this tool proactively in these scenarios:

1. Complex multi-step tasks - When a task requires 3 or more distinct steps or actions
2. Non-trivial and complex tasks - Tasks that require careful planning or multiple operations
3. User explicitly requests todo list - When the user directly asks you to use the todo list
4. User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)
5. After receiving new instructions - Immediately capture user requirements as todos
6. When you start working on a task - Mark it as in_progress BEFORE beginning work. Ideally you should only have one todo as in_progress at a time
7. After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation

## When NOT to Use This Tool

Skip using this tool when:
1. There is only a single, straightforward task
2. The task is trivial and tracking it provides no organizational benefit
3. The task can be completed in less than 3 trivial steps
4. The task is purely conversational or informational

NOTE that you should not use this tool if there is only one trivial task to do. In this case you are better off just doing the task directly.

## Examples of When to Use the Todo List

<example>
User: I want to add a dark mode toggle to the application settings. Make sure you run the tests and build when you're done!
Assistant: *Creates todo list with the following items:*
1. Creating dark mode toggle component in Settings page
2. Adding dark mode state management (context/store)
3. Implementing CSS-in-JS styles for dark theme
4. Updating existing components to support theme switching
5. Running tests and build process, addressing any failures or errors that occur
*Begins working on the first task*

<reasoning>
The assistant used the todo list because:
1. Adding dark mode is a multi-step feature requiring UI, state management, and styling changes
2. The user explicitly requested tests and build be run afterward
3. The assistant inferred that tests and build need to pass by adding "Ensure tests and build succeed" as the final task
</reasoning>
</example>

<example>
User: Help me rename the function getCwd to getCurrentWorkingDirectory across my project
Assistant: *Uses grep or search tools to locate all instances of getCwd in the codebase*
I've found 15 instances of 'getCwd' across 8 different files.
*Creates todo list with specific items for each file that needs updating*

<reasoning>
The assistant used the todo list because:
1. First, the assistant searched to understand the scope of the task
2. Upon finding multiple occurrences across different files, it determined this was a complex task with multiple steps
3. The todo list helps ensure every instance is tracked and updated systematically
4. This approach prevents missing any occurrences and maintains code consistency
</reasoning>
</example>


<example>
User: I need to implement these features for my e-commerce site: user registration, product catalog, shopping cart, and checkout flow.
Assistant: *Creates a todo list breaking down each feature into specific tasks based on the project architecture*
Let's start with user registration. This will involve creating database models, API endpoints, and frontend forms that integrate with your existing authentication system.

<reasoning>
The assistant used the todo list because:
1. The user provided multiple complex features to implement in a comma separated list
2. The todo list helps organize these large features into manageable tasks
3. This approach allows for tracking progress across the entire implementation
</reasoning>
</example>

<example>
User: Can you help optimize my React application? It's rendering slowly and has performance issues.
Assistant: *Reviews component structure, render patterns, state management, and data fetching*
After analyzing your codebase, I've identified several performance issues.
*Creates todo list with items like: 1) Implementing memoization for expensive calculations in ProductList, 2) Adding virtualization for long lists in Dashboard, 3) Optimizing image loading in Gallery component, 4) Fixing state update loops in ShoppingCart, 5) Reviewing bundle size and implementing code splitting*
Let's start by implementing memoization for the expensive calculations in your ProductList component.

<reasoning>
The assistant used the todo list because:
1. First, the assistant examined the codebase to identify specific performance issues
2. Based on this analysis, it identified multiple optimization opportunities
3. Performance optimization is a non-trivial task requiring multiple steps
4. The todo list helps methodically track improvements across different components
5. This systematic approach ensures all performance bottlenecks are addressed
</reasoning>
</example>

## Examples of When NOT to Use the Todo List

<example>
User: How do I print 'Hello World' in Python?
Assistant: In Python, you can print "Hello World" with this simple code:

python
print("Hello World")

This will output the text "Hello World" to the console when executed.

<reasoning>
The assistant did not use the todo list because this is a single, trivial task that can be completed in one step. There's no need to track multiple tasks or steps for such a straightforward request.
</reasoning>
</example>

<example>
User: What does the git status command do?
Assistant: The git status command shows the current state of your working directory and staging area. It displays which changes have been staged, which haven't, and which files aren't being tracked by Git. This is a fundamental command for understanding the current state of your repository before making commits.

<reasoning>
The assistant did not use the todo list because this is an informational request with no actual coding task to complete. The user is simply asking for an explanation, not for the assistant to perform multiple steps or tasks.
</reasoning>
</example>

<example>
User: Can you add a comment to the calculateTotal function to explain what it does?
Assistant: Sure, let me add a comment to the calculateTotal function to explain what it does.
* Uses the Edit tool to add a comment to the calculateTotal function *

<reasoning>
The assistant did not use the todo list because this is a single, straightforward task confined to one location in the code. Adding a comment doesn't require tracking multiple steps or systematic organization.
</reasoning>
</example>

<example>
User: Run npm install for me and tell me what happens.
Assistant: I'll run the npm install command for you.

*Executes: npm install*

The command completed successfully. Here's the output:
[Output of npm install command]

All dependencies have been installed according to your package.json file.

<reasoning>
The assistant did not use the todo list because this is a single command execution with immediate results. There are no multiple steps to track or organize, making the todo list unnecessary for this straightforward task.
</reasoning>
</example>

## Task States and Management

1. **Task States**: Use these states to track progress:
   - pending: Task not yet started
   - in_progress: Currently working on (limit to ONE task at a time)
   - completed: Task finished successfully

   **IMPORTANT**: Task descriptions must have two forms:
   - content: The imperative form describing what needs to be done (e.g., "Run tests", "Build the project")
   - activeForm: The present continuous form shown during execution (e.g., "Running tests", "Building the project")

2. **Task Management**:
   - Update task status in real-time as you work
   - Mark tasks complete IMMEDIATELY after finishing (don't batch completions)
   - Exactly ONE task must be in_progress at any time (not less, not more)
   - Complete current tasks before starting new ones
   - Remove tasks that are no longer relevant from the list entirely

3. **Task Completion Requirements**:
   - ONLY mark a task as completed when you have FULLY accomplished it
   - If you encounter errors, blockers, or cannot finish, keep the task as in_progress
   - When blocked, create a new task describing what needs to be resolved
   - Never mark a task as completed if:
     - Tests are failing
     - Implementation is partial
     - You encountered unresolved errors
     - You couldn't find necessary files or dependencies

4. **Task Breakdown**:
   - Create specific, actionable items
   - Break complex tasks into smaller, manageable steps
   - Use clear, descriptive task names
   - Always provide both forms:
     - content: "Fix authentication bug"
     - activeForm: "Fixing authentication bug"

When in doubt, use this tool. Being proactive with task management demonstrates attentiveness and ensures you complete all requirements successfully.
```

注意几个看起来像但不是模板占位的位置：

- `${FILE_EDIT_TOOL_NAME}` 是源码里的 JS 表达式，在文件被 import 时已经被求值；模型看到的就是字面量 `Edit`。
- 其它所有 `${...}` 形态的子串在 `PROMPT` 源代码中并不存在；如果你在生成的 API 请求里看到任何带 `${` 的内容，说明上游构建工具没替换成功。

### TaskCreate

来源：`src/tools/TaskCreateTool/prompt.ts` 的 `getPrompt()`。该函数有两条分支：

```typescript
const teammateContext = isAgentSwarmsEnabled()
  ? ' and potentially assigned to teammates'
  : ''

const teammateTips = isAgentSwarmsEnabled()
  ? `- Include enough detail in the description for another agent to understand and complete the task
- New tasks are created with status 'pending' and no owner - use TaskUpdate with the \`owner\` parameter to assign them
`
  : ''
```

`teammateContext` 插入第二段的复杂任务条目末尾；`teammateTips` 插入 Tips 段中间。

#### 非 swarm 模式（外部用户默认）

```text
Use this tool to create a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.
It also helps the user understand the progress of the task and overall progress of their requests.

## When to Use This Tool

Use this tool proactively in these scenarios:

- Complex multi-step tasks - When a task requires 3 or more distinct steps or actions
- Non-trivial and complex tasks - Tasks that require careful planning or multiple operations
- Plan mode - When using plan mode, create a task list to track the work
- User explicitly requests todo list - When the user directly asks you to use the todo list
- User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)
- After receiving new instructions - Immediately capture user requirements as tasks
- When you start working on a task - Mark it as in_progress BEFORE beginning work
- After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation

## When NOT to Use This Tool

Skip using this tool when:
- There is only a single, straightforward task
- The task is trivial and tracking it provides no organizational benefit
- The task can be completed in less than 3 trivial steps
- The task is purely conversational or informational

NOTE that you should not use this tool if there is only one trivial task to do. In this case you are better off just doing the task directly.

## Task Fields

- **subject**: A brief, actionable title in imperative form (e.g., "Fix authentication bug in login flow")
- **description**: What needs to be done
- **activeForm** (optional): Present continuous form shown in the spinner when the task is in_progress (e.g., "Fixing authentication bug"). If omitted, the spinner shows the subject instead.

All tasks are created with status `pending`.

## Tips

- Create tasks with clear, specific subjects that describe the outcome
- After creating tasks, use TaskUpdate to set up dependencies (blocks/blockedBy) if needed
- Check TaskList first to avoid creating duplicate tasks
```

#### swarm 模式（ant 用户、或 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` + `tengu_amber_flint`）

只列差异行：

```text
...
- Non-trivial and complex tasks - Tasks that require careful planning or multiple operations and potentially assigned to teammates
- Plan mode - When using plan mode, create a task list to track the work
...
## Tips

- Create tasks with clear, specific subjects that describe the outcome
- After creating tasks, use TaskUpdate to set up dependencies (blocks/blockedBy) if needed
- Include enough detail in the description for another agent to understand and complete the task
- New tasks are created with status 'pending' and no owner - use TaskUpdate with the `owner` parameter to assign them
- Check TaskList first to avoid creating duplicate tasks
```

### TaskUpdate

来源：`src/tools/TaskUpdateTool/prompt.ts` 的 `PROMPT` 常量。无分支。

```text
Use this tool to update a task in the task list.

## When to Use This Tool

**Mark tasks as resolved:**
- When you have completed the work described in a task
- When a task is no longer needed or has been superseded
- IMPORTANT: Always mark your assigned tasks as resolved when you finish them
- After resolving, call TaskList to find your next task

- ONLY mark a task as completed when you have FULLY accomplished it
- If you encounter errors, blockers, or cannot finish, keep the task as in_progress
- When blocked, create a new task describing what needs to be resolved
- Never mark a task as completed if:
  - Tests are failing
  - Implementation is partial
  - You encountered unresolved errors
  - You couldn't find necessary files or dependencies

**Delete tasks:**
- When a task is no longer relevant or was created in error
- Setting status to `deleted` permanently removes the task

**Update task details:**
- When requirements change or become clearer
- When establishing dependencies between tasks

## Fields You Can Update

- **status**: The task status (see Status Workflow below)
- **subject**: Change the task title (imperative form, e.g., "Run tests")
- **description**: Change the task description
- **activeForm**: Present continuous form shown in spinner when in_progress (e.g., "Running tests")
- **owner**: Change the task owner (agent name)
- **metadata**: Merge metadata keys into the task (set a key to null to delete it)
- **addBlocks**: Mark tasks that cannot start until this one completes
- **addBlockedBy**: Mark tasks that must complete before this one can start

## Status Workflow

Status progresses: `pending` → `in_progress` → `completed`

Use `deleted` to permanently remove a task.

## Staleness

Make sure to read a task's latest state using `TaskGet` before updating it.

## Examples

Mark task as in progress when starting work:
```json
{"taskId": "1", "status": "in_progress"}
```

Mark task as completed after finishing work:
```json
{"taskId": "1", "status": "completed"}
```

Delete a task:
```json
{"taskId": "1", "status": "deleted"}
```

Claim a task by setting owner:
```json
{"taskId": "1", "owner": "my-name"}
```

Set up task dependencies:
```json
{"taskId": "2", "addBlockedBy": ["1"]}
```
```

注意：

- 文本里同时提到 `owner` 和 swarm 相关用法，但 prompt 本身不按 swarm 分支裁剪 —— 也就是说，即使外部用户没启用 swarm，模型看到的 TaskUpdate prompt 也会鼓励 `owner` 字段；`input_schema` 也保留 `owner`。
- `addBlocks` / `addBlockedBy` 是写动作；prompt 没有提供 `removeBlocks` / `removeBlockedBy`，对应 `input_schema` 也没这两个字段。

### TaskGet

来源：`src/tools/TaskGetTool/prompt.ts` 的 `PROMPT` 常量。无分支。

```text
Use this tool to retrieve a task by its ID from the task list.

## When to Use This Tool

- When you need the full description and context before starting work on a task
- To understand task dependencies (what it blocks, what blocks it)
- After being assigned a task, to get complete requirements

## Output

Returns full task details:
- **subject**: Task title
- **description**: Detailed requirements and context
- **status**: 'pending', 'in_progress', or 'completed'
- **blocks**: Tasks waiting on this one to complete
- **blockedBy**: Tasks that must complete before this one can start

## Tips

- After fetching a task, verify its blockedBy list is empty before beginning work.
- Use TaskList to see all tasks in summary form.
```

### TaskList

来源：`src/tools/TaskListTool/prompt.ts` 的 `getPrompt()`。该函数按 `isAgentSwarmsEnabled()` 拼三个变量（`teammateUseCase` / `idDescription` / `teammateWorkflow`）。其中 `idDescription` 两个分支的实际字符串相同，所以非 swarm 与 swarm 的差异等价于：插入 `teammateUseCase` 一行 + 末尾整段 `Teammate Workflow`。

#### 非 swarm 模式

```text
Use this tool to list all tasks in the task list.

## When to Use This Tool

- To see what tasks are available to work on (status: 'pending', no owner, not blocked)
- To check overall progress on the project
- To find tasks that are blocked and need dependencies resolved
- After completing a task, to check for newly unblocked work or claim the next available task
- **Prefer working on tasks in ID order** (lowest ID first) when multiple tasks are available, as earlier tasks often set up context for later ones

## Output

Returns a summary of each task:
- **id**: Task identifier (use with TaskGet, TaskUpdate)
- **subject**: Brief description of the task
- **status**: 'pending', 'in_progress', or 'completed'
- **owner**: Agent ID if assigned, empty if available
- **blockedBy**: List of open task IDs that must be resolved first (tasks with blockedBy cannot be claimed until dependencies resolve)

Use TaskGet with a specific task ID to view full details including description and comments.

```

#### swarm 模式

只列差异：

```text
...
- To find tasks that are blocked and need dependencies resolved
- Before assigning tasks to teammates, to see what's available
- After completing a task, to check for newly unblocked work or claim the next available task
...
Use TaskGet with a specific task ID to view full details including description and comments.

## Teammate Workflow

When working as a teammate:
1. After completing your current task, call TaskList to find available work
2. Look for tasks with status 'pending', no owner, and empty blockedBy
3. **Prefer tasks in ID order** (lowest ID first) when multiple tasks are available, as earlier tasks often set up context for later ones
4. Claim an available task using TaskUpdate (set `owner` to your name), or wait for leader assignment
5. If blocked, focus on unblocking tasks or notify the team lead
```

## `description()` 短摘要（不进 API tools[]，仅 ToolSearch / UI 用）

```text
TodoWrite  Update the todo list for the current session. To be used proactively and often to track progress and pending tasks. Make sure that at least one task is in_progress at all times. Always provide both content (imperative) and activeForm (present continuous) for each task.
TaskCreate Create a new task in the task list
TaskUpdate Update a task in the task list
TaskGet    Get a task by ID from the task list
TaskList   List all tasks in the task list
```

这些字符串只在 `tool.description()` 被调用的路径里出现，例如 ToolSearch 工具列出 deferred tool 的 `searchHint`/`description`，或终端 UI 的标题。它们**不**会作为字段进入 Messages API 请求体。

## `input_schema` 字段（与 prompt 同时拼装）

`input_schema` 由 `zodToJsonSchema(tool.inputSchema)` 生成。下面是去掉冗余 metadata 后的简化形态。

### TodoWrite

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "todos": {
      "type": "array",
      "description": "The updated todo list",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "content": { "type": "string" },
          "activeForm": { "type": "string" },
          "status": {
            "type": "string",
            "enum": ["pending", "in_progress", "completed"]
          }
        },
        "required": ["content", "activeForm", "status"]
      }
    }
  },
  "required": ["todos"]
}
```

外层包装会标 `strict: true`。

### TaskCreate

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "subject": { "type": "string", "description": "A brief title for the task" },
    "description": { "type": "string", "description": "What needs to be done" },
    "activeForm": {
      "type": "string",
      "description": "Present continuous form shown in spinner when in_progress (e.g., \"Running tests\")"
    },
    "metadata": {
      "type": "object",
      "description": "Arbitrary metadata to attach to the task",
      "additionalProperties": {}
    }
  },
  "required": ["subject", "description"]
}
```

### TaskUpdate

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "taskId": { "type": "string", "description": "The ID of the task to update" },
    "subject": { "type": "string", "description": "New subject for the task" },
    "description": { "type": "string", "description": "New description for the task" },
    "activeForm": {
      "type": "string",
      "description": "Present continuous form shown in spinner when in_progress (e.g., \"Running tests\")"
    },
    "status": {
      "type": "string",
      "description": "New status for the task",
      "enum": ["pending", "in_progress", "completed", "deleted"]
    },
    "addBlocks": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Task IDs that this task blocks"
    },
    "addBlockedBy": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Task IDs that block this task"
    },
    "owner": { "type": "string", "description": "New owner for the task" },
    "metadata": {
      "type": "object",
      "description": "Metadata keys to merge into the task. Set a key to null to delete it.",
      "additionalProperties": {}
    }
  },
  "required": ["taskId"]
}
```

### TaskGet

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "taskId": { "type": "string", "description": "The ID of the task to retrieve" }
  },
  "required": ["taskId"]
}
```

### TaskList

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {}
}
```

## 一个完整 entry 在 API 请求里的样子

以 TaskCreate（非 swarm、未触发 strict / fgts / defer）为例，序列化进 `tools[]` 后的单条 entry 实际形态：

```json
{
  "name": "TaskCreate",
  "description": "Use this tool to create a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.\nIt also helps the user understand the progress of the task and overall progress of their requests.\n\n## When to Use This Tool\n\nUse this tool proactively in these scenarios:\n\n- Complex multi-step tasks - When a task requires 3 or more distinct steps or actions\n- Non-trivial and complex tasks - Tasks that require careful planning or multiple operations\n- Plan mode - When using plan mode, create a task list to track the work\n- User explicitly requests todo list - When the user directly asks you to use the todo list\n- User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)\n- After receiving new instructions - Immediately capture user requirements as tasks\n- When you start working on a task - Mark it as in_progress BEFORE beginning work\n- After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation\n\n## When NOT to Use This Tool\n\nSkip using this tool when:\n- There is only a single, straightforward task\n- The task is trivial and tracking it provides no organizational benefit\n- The task can be completed in less than 3 trivial steps\n- The task is purely conversational or informational\n\nNOTE that you should not use this tool if there is only one trivial task to do. In this case you are better off just doing the task directly.\n\n## Task Fields\n\n- **subject**: A brief, actionable title in imperative form (e.g., \"Fix authentication bug in login flow\")\n- **description**: What needs to be done\n- **activeForm** (optional): Present continuous form shown in the spinner when the task is in_progress (e.g., \"Fixing authentication bug\"). If omitted, the spinner shows the subject instead.\n\nAll tasks are created with status `pending`.\n\n## Tips\n\n- Create tasks with clear, specific subjects that describe the outcome\n- After creating tasks, use TaskUpdate to set up dependencies (blocks/blockedBy) if needed\n- Check TaskList first to avoid creating duplicate tasks\n",
  "input_schema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "subject": { "type": "string", "description": "A brief title for the task" },
      "description": { "type": "string", "description": "What needs to be done" },
      "activeForm": {
        "type": "string",
        "description": "Present continuous form shown in spinner when in_progress (e.g., \"Running tests\")"
      },
      "metadata": {
        "type": "object",
        "description": "Arbitrary metadata to attach to the task",
        "additionalProperties": {}
      }
    },
    "required": ["subject", "description"]
  }
}
```

`description` 字段就是把上一节的 TaskCreate prompt 整段以 JSON 字符串形式塞进去（包含换行 `\n` 与 markdown 标题）。模型在每个 API turn 都会重新读取这段文本，因此它实际上和 system prompt 一样承担引导职责，只是按工具维度归档。

## 几个容易踩的坑

1. **工具调用顺序约束写在 prompt 里，不在 schema 里**。TaskUpdate 的"Staleness"段落要求先 `TaskGet` 再 `TaskUpdate`，但 `input_schema` 不能强制这点；模型可能跳过 `TaskGet`，runtime 不会拒绝。
2. **`activeForm` 在 TaskCreate 是 optional，在 TodoWrite 的 item 里是 required**。两个工具对同名字段的强制等级不同；mocks 时容易写错。
3. **`status: "deleted"` 只在 TaskUpdate prompt 和 schema 里出现**。`TaskCreate` 不接受 `deleted`，`TaskList` / `TaskGet` 的输出也不会出现 `deleted` 状态 —— 它在文件层面就是删除文件。
4. **TodoWrite 的 prompt 里有一处空 example**：第二个 example 后面跟了两个换行再接下一个 `<example>`，源码里就是这样。这是模板字面量的字面内容，不是 bug。
5. **prompt 的字面 `${FILE_EDIT_TOOL_NAME}` 只在 TodoWrite/prompt.ts 中存在**，其它 4 个工具的 prompt 不含任何运行时 substitution。所以五个工具中只有 TodoWrite 的 prompt 字符串依赖 import 顺序。
6. **缓存命中后 swarm 翻转无效**。如果你在调试时 toggle `USER_TYPE=ant` 想让 TaskCreate 切到 swarm 文案，必须重启进程让 `getToolSchemaCache()` 重新拉。

## 阅读源码定位

| 想确认的事 | 看 |
|------------|-----|
| `description` 字段实际从哪里来 | `src/utils/api.ts` 的 `toolToAPISchema()` 第 169-178 行 |
| 缓存策略与命中条件 | `src/utils/api.ts:147` 附近 `cacheKey` 计算 + `getToolSchemaCache()` |
| TodoWrite prompt 原文 | `src/tools/TodoWriteTool/prompt.ts` 的 `PROMPT` |
| TaskCreate prompt 与 swarm 分支 | `src/tools/TaskCreateTool/prompt.ts` 的 `getPrompt()` |
| TaskUpdate prompt 原文 | `src/tools/TaskUpdateTool/prompt.ts` 的 `PROMPT` |
| TaskGet prompt 原文 | `src/tools/TaskGetTool/prompt.ts` 的 `PROMPT` |
| TaskList prompt 与 swarm 分支 | `src/tools/TaskListTool/prompt.ts` 的 `getPrompt()` |
| swarm 是否开启 | `src/utils/agentSwarmsEnabled.ts` 的 `isAgentSwarmsEnabled()` |
| FILE_EDIT_TOOL_NAME 字面值 | `src/tools/FileEditTool/constants.ts` |
