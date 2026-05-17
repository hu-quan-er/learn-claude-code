# 03 - 工具传递与 tool_use 决策链路

这一篇回答一个关键问题：`TodoWrite`、`TaskCreate`、`TaskUpdate`、`TaskGet`、`TaskList` 这些工具是以什么形式传给模型的？模型又如何基于这些定义返回 `tool_use`？

## 一句话流程

```text
本地 Tool 对象
  → toolToAPISchema()
  → Anthropic Messages API 的 tools 数组
  → 模型选择输出 text 或 tool_use block
  → query.ts 捕获 tool_use
  → StreamingToolExecutor / toolExecution 执行本地工具
  → runtime 生成 user/tool_result block
  → 下一轮请求继续发给模型
```

所以工具不是作为普通自然语言“贴进用户消息”的，标准路径是作为 Messages API 请求体里的 `tools` 数组传给模型。每个工具给模型看的核心字段是：

```json
{
  "name": "TaskUpdate",
  "description": "...tool.prompt() 生成的说明...",
  "input_schema": {
    "type": "object",
    "properties": {
      "taskId": { "type": "string" },
      "status": {
        "type": "string",
        "enum": ["pending", "in_progress", "completed", "deleted"]
      }
    },
    "required": ["taskId"]
  }
}
```

## 第 1 步：工具先作为本地 Tool 对象注册

**文件**：`src/tools.ts`

`getAllBaseTools()` 是基础工具集合的来源。Todo 相关工具的注册点是：

```typescript
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // ...
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    // ...
    ...(isTodoV2Enabled()
      ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool]
      : []),
    // ...
  ]
}
```

这里有两个层次：

| 层次 | 说明 |
|------|------|
| 注册 | `TodoWriteTool` 总是在基础工具集合里；V2 开启时追加 `Task*` 工具 |
| 启用 | 单个工具还有自己的 `isEnabled()`，例如 `TodoWrite` 要求 `!isTodoV2Enabled()`，`Task*` 要求 `isTodoV2Enabled()` |

本地 `Tool` 对象包含的信息比模型看到的更多，例如：

```text
name
prompt()
inputSchema
call()
checkPermissions()
mapToolResultToToolResultBlockParam()
isEnabled()
isReadOnly()
isConcurrencySafe()
shouldDefer
renderToolUseMessage()
```

模型不会直接看到 `call()` 或 `checkPermissions()` 的实现。模型只看到 API schema 和 prompt 描述；执行逻辑留在本地 runtime。

## 第 2 步：Tool 转成 API schema

**文件**：`src/utils/api.ts`

`toolToAPISchema()` 是转换入口：

```typescript
base = {
  name: tool.name,
  description: await tool.prompt({
    getToolPermissionContext: options.getToolPermissionContext,
    tools: options.tools,
    agents: options.agents,
    allowedAgentTypes: options.allowedAgentTypes,
  }),
  input_schema,
}
```

这里有几个重要点：

- `name` 来自工具对象的 `tool.name`，例如 `TaskUpdate`。
- `description` 来自 `tool.prompt(...)`，不是 `tool.description(...)`。
- `input_schema` 优先使用工具自带的 `inputJSONSchema`；否则通过 `zodToJsonSchema(tool.inputSchema)` 从 Zod schema 转成 JSON Schema。
- 如果工具声明 `strict: true`，且当前模型支持 structured outputs，schema 上会额外带 `strict: true`。
- 如果启用 fine-grained tool streaming，schema 上可能带 `eager_input_streaming: true`。
- 如果动态工具加载启用，schema 上可能带 `defer_loading: true`。

对 todo 工具来说，模型最终看到的是类似下面的工具定义。

### TodoWrite 的 API 形态

```json
{
  "name": "TodoWrite",
  "description": "Use this tool to create and manage a structured task list...",
  "input_schema": {
    "type": "object",
    "properties": {
      "todos": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "content": { "type": "string" },
            "status": {
              "type": "string",
              "enum": ["pending", "in_progress", "completed"]
            },
            "activeForm": { "type": "string" }
          },
          "required": ["content", "status", "activeForm"]
        }
      }
    },
    "required": ["todos"]
  },
  "strict": true
}
```

### TaskUpdate 的 API 形态

```json
{
  "name": "TaskUpdate",
  "description": "Update a task...",
  "input_schema": {
    "type": "object",
    "properties": {
      "taskId": { "type": "string" },
      "subject": { "type": "string" },
      "description": { "type": "string" },
      "activeForm": { "type": "string" },
      "status": {
        "type": "string",
        "enum": ["pending", "in_progress", "completed", "deleted"]
      },
      "addBlocks": {
        "type": "array",
        "items": { "type": "string" }
      },
      "addBlockedBy": {
        "type": "array",
        "items": { "type": "string" }
      },
      "owner": { "type": "string" },
      "metadata": { "type": "object" }
    },
    "required": ["taskId"]
  }
}
```

上面的 JSON 是按源码 schema 还原出的简化形态；真实请求里还会受 feature gate、模型能力、tool search、cache control 等逻辑影响。

## 第 3 步：API 请求携带 tools 数组

**文件**：`src/services/api/claude.ts`

请求构建前会先处理动态工具加载，然后生成工具 schema：

```typescript
const toolSchemas = await Promise.all(
  filteredTools.map(tool =>
    toolToAPISchema(tool, {
      getToolPermissionContext: options.getToolPermissionContext,
      tools,
      agents: options.agents,
      allowedAgentTypes: options.allowedAgentTypes,
      model: options.model,
      deferLoading: willDefer(tool),
    }),
  ),
)

const allTools = [...toolSchemas, ...extraToolSchemas]
```

最后请求参数包含：

```typescript
return {
  model: normalizeModelStringForAPI(options.model),
  messages: addCacheBreakpoints(...),
  system,
  tools: allTools,
  tool_choice: options.toolChoice,
  metadata: getAPIMetadata(),
  max_tokens: maxOutputTokens,
  thinking,
}
```

简化成 API JSON，大致是：

```json
{
  "model": "claude-...",
  "system": "...系统提示...",
  "messages": [
    { "role": "user", "content": "请帮我实现登录页并运行测试" }
  ],
  "tools": [
    {
      "name": "TaskCreate",
      "description": "...",
      "input_schema": { "type": "object", "properties": {} }
    },
    {
      "name": "TaskUpdate",
      "description": "...",
      "input_schema": { "type": "object", "properties": {} }
    }
  ],
  "tool_choice": null
}
```

正常主查询里，`tool_choice` 通常是 `undefined` / auto 语义：模型可以自己决定回复文本，也可以返回一个或多个 `tool_use`。一些 side query 或特定流程可以显式指定 `tool_choice`，强制模型调用某个工具。

## 第 4 步：动态工具加载下的 deferred 形态

Todo 相关工具都声明了 `shouldDefer: true`。这不代表它们一定不会发给模型，而是说在 tool search 启用时，它们可能走动态加载路径。

**文件**：`src/tools/ToolSearchTool/prompt.ts`

```typescript
export function isDeferredTool(tool: Tool): boolean {
  if (tool.alwaysLoad === true) return false
  if (tool.isMcp === true) return true
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false
  return tool.shouldDefer === true
}

export function formatDeferredToolLine(tool: Tool): string {
  return tool.name
}
```

**文件**：`src/services/api/claude.ts`

如果 tool search 启用，会先计算 deferred 工具名：

```typescript
const deferredToolNames = new Set<string>()
if (useToolSearch) {
  for (const t of tools) {
    if (isDeferredTool(t)) deferredToolNames.add(t.name)
  }
}
```

然后过滤实际发送 schema 的工具：

```typescript
filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true
  if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true
  return discoveredToolNames.has(tool.name)
})
```

如果 deferred tool 尚未被发现，模型可能先只看到一个 meta message：

```xml
<available-deferred-tools>
TaskCreate
TaskGet
TaskList
TaskUpdate
</available-deferred-tools>
```

此时完整 `input_schema` 不一定已经在 `tools` 数组里。模型需要先调用 `ToolSearch` 发现对应工具，后续请求才会把完整 schema 带上。`toolExecution.ts` 里还有兜底逻辑：如果模型在 schema 未发送时直接调用 deferred 工具并导致 Zod 校验失败，会在错误结果中提示先 `ToolSearch("select:<tool>")` 再重试。

因此可以把 todo 工具传递分成两种模式：

| 模式 | 模型看到什么 |
|------|--------------|
| 标准工具模式 | `tools` 数组里直接包含 `TodoWrite` 或 `Task*` 的完整 schema |
| ToolSearch 动态模式 | 起初可能只在 `<available-deferred-tools>` 里看到工具名；发现后才在 `tools` 数组里看到完整 schema |

## 第 5 步：模型返回 tool_use

当模型决定使用任务工具时，它返回的是 assistant message 的 content block，而不是自然语言中的伪 JSON。

V1 示例：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_001",
      "name": "TodoWrite",
      "input": {
        "todos": [
          {
            "content": "Implement login form UI",
            "activeForm": "Implementing login form UI",
            "status": "in_progress"
          },
          {
            "content": "Run tests for login flow",
            "activeForm": "Running tests for login flow",
            "status": "pending"
          }
        ]
      }
    }
  ]
}
```

V2 示例：

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_101",
      "name": "TaskUpdate",
      "input": {
        "taskId": "1",
        "status": "completed"
      }
    }
  ]
}
```

这个 `input` 必须符合前面传给模型的 `input_schema`。但 runtime 仍然会用工具自己的 Zod schema 再校验一次，因为模型可能生成不合法参数。

## 第 6 步：query 捕获 tool_use 并交给执行器

**文件**：`src/query.ts`

流式读取 assistant message 时，`query.ts` 会提取所有 `tool_use` block：

```typescript
const msgToolUseBlocks = message.message.content.filter(
  content => content.type === 'tool_use',
) as ToolUseBlock[]

if (msgToolUseBlocks.length > 0) {
  toolUseBlocks.push(...msgToolUseBlocks)
  needsFollowUp = true
}
```

如果启用了 streaming tool execution，会立即把工具调用加入执行器：

```typescript
for (const toolBlock of msgToolUseBlocks) {
  streamingToolExecutor.addTool(toolBlock, message)
}
```

随后 `query.ts` 会不断读取执行完成的结果：

```typescript
for (const result of streamingToolExecutor.getCompletedResults()) {
  if (result.message) {
    yield result.message
    toolResults.push(
      ...normalizeMessagesForAPI([result.message], tools).filter(
        _ => _.type === 'user',
      ),
    )
  }
}
```

这里的关键点是：`tool_result` 在 Messages API 语义里是一个 `user` role message block，用来回应刚才 assistant 的 `tool_use_id`。

## 第 7 步：执行器校验输入、权限、调用工具

**文件**：`src/services/tools/StreamingToolExecutor.ts`

工具入队时先查找本地工具定义：

```typescript
const toolDefinition = findToolByName(this.toolDefinitions, block.name)
```

找不到会直接生成错误 `tool_result`：

```json
{
  "type": "tool_result",
  "is_error": true,
  "tool_use_id": "toolu_101",
  "content": "<tool_use_error>Error: No such tool available: TaskUpdate</tool_use_error>"
}
```

找到后，先用本地 `inputSchema` 判断并发安全性：

```typescript
const parsedInput = toolDefinition.inputSchema.safeParse(block.input)
const isConcurrencySafe = parsedInput?.success
  ? Boolean(toolDefinition.isConcurrencySafe(parsedInput.data))
  : false
```

**文件**：`src/services/tools/toolExecution.ts`

真正执行前再次用 Zod 校验：

```typescript
const parsedInput = tool.inputSchema.safeParse(input)
if (!parsedInput.success) {
  return [
    {
      message: createUserMessage({
        content: [
          {
            type: 'tool_result',
            content: `<tool_use_error>InputValidationError: ...</tool_use_error>`,
            is_error: true,
            tool_use_id: toolUseID,
          },
        ],
      }),
    },
  ]
}
```

校验后还会经过：

```text
validateInput?
  → PreToolUse hooks
  → checkPermissions()
  → 用户批准 / 拒绝 / 自动允许
  → tool.call()
```

Todo 工具的权限很轻：

| 工具 | 权限策略 |
|------|----------|
| `TodoWrite` | `checkPermissions()` 直接 `{ behavior: 'allow' }` |
| `TaskCreate` / `TaskUpdate` / `TaskGet` / `TaskList` | 任务工具自身实现中也不走文件编辑那种显式用户确认；主要风险控制来自 schema、hooks、状态逻辑 |

执行本体是：

```typescript
const result = await tool.call(
  callInput,
  {
    ...toolUseContext,
    toolUseId: toolUseID,
    userModified: permissionDecision.userModified ?? false,
  },
  canUseTool,
  assistantMessage,
  progress => {
    onToolProgress({
      toolUseID: progress.toolUseID,
      data: progress.data,
    })
  },
)
```

对 todo 来说，这一步分别落到：

| tool_use.name | 本地执行 |
|---------------|----------|
| `TodoWrite` | `TodoWriteTool.call()` 写 `AppState.todos[todoKey]` |
| `TaskCreate` | `TaskCreateTool.call()` 创建 `{id}.json` |
| `TaskUpdate` | `TaskUpdateTool.call()` 合并更新任务文件、依赖、owner、status |
| `TaskGet` | `TaskGetTool.call()` 读取单个任务 |
| `TaskList` | `TaskListTool.call()` 列出任务 |

## 第 8 步：工具结果变成 tool_result 回填给模型

工具 `call()` 返回的内部数据不会原样裸塞回 API，而是先通过工具自己的 mapper：

```typescript
const mappedToolResultBlock = tool.mapToolResultToToolResultBlockParam(
  result.data,
  toolUseID,
)
```

随后创建 user message：

```typescript
createUserMessage({
  content: [
    {
      type: 'tool_result',
      tool_use_id: toolUseID,
      content: 'Updated task #1 status',
    },
  ],
  toolUseResult: toolUseResult,
  sourceToolAssistantUUID: assistantMessage.uuid,
})
```

API 形态大致是：

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_101",
      "content": "Updated task #1 status"
    }
  ]
}
```

`tool_use_id` 必须对应前面 assistant `tool_use.id`。这就是模型能知道“哪个工具调用得到了哪个执行结果”的原因。

对于 todo 工具，常见 result 文本是：

| 工具 | 成功 result 文本 |
|------|------------------|
| `TodoWrite` | `Todos have been modified successfully...` |
| `TaskCreate` | `Task #<id> created successfully: <subject>` |
| `TaskUpdate` | `Updated task #<taskId> <updatedFields...>` |
| `TaskGet` | `Task #<id>: <subject> ...` 或 `Task not found` |
| `TaskList` | 多行 `#<id> [status] subject` 或 `No tasks found` |

## 第 9 步：下一轮模型继续基于结果决策

工具调用并不是一次请求里“模型自己执行”的。真实循环是：

```text
Round N request:
  user asks for work
  tools schema includes TaskCreate / TaskUpdate

Round N response:
  assistant returns tool_use(TaskCreate)

local runtime:
  executes TaskCreate
  writes task JSON
  creates user/tool_result

Round N+1 request:
  messages now include:
    assistant/tool_use(TaskCreate)
    user/tool_result("Task #1 created successfully...")
  tools schema still includes available tools

Round N+1 response:
  assistant may call TaskUpdate, TaskList, or continue with text
```

所以模型的“决策”发生在每轮 API 响应中；Claude Code runtime 负责把工具执行结果补成下一轮上下文。

## 以 TaskCreate 为例的完整请求响应

### 请求

```json
{
  "messages": [
    {
      "role": "user",
      "content": "请帮我实现登录页，包含校验并运行测试"
    }
  ],
  "tools": [
    {
      "name": "TaskCreate",
      "description": "...",
      "input_schema": {
        "type": "object",
        "properties": {
          "subject": { "type": "string" },
          "description": { "type": "string" },
          "activeForm": { "type": "string" },
          "metadata": { "type": "object" }
        },
        "required": ["subject", "description"]
      }
    },
    {
      "name": "TaskUpdate",
      "description": "...",
      "input_schema": {
        "type": "object",
        "properties": {
          "taskId": { "type": "string" },
          "status": { "type": "string" }
        },
        "required": ["taskId"]
      }
    }
  ]
}
```

### 模型响应

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_create_1",
      "name": "TaskCreate",
      "input": {
        "subject": "Implement login form UI",
        "description": "Create the login page form fields and layout",
        "activeForm": "Implementing login form UI"
      }
    }
  ]
}
```

### Runtime 执行后追加的 tool_result

```json
{
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "toolu_create_1",
      "content": "Task #1 created successfully: Implement login form UI"
    }
  ]
}
```

### 下一轮模型继续决策

```json
{
  "role": "assistant",
  "content": [
    {
      "type": "tool_use",
      "id": "toolu_update_1",
      "name": "TaskUpdate",
      "input": {
        "taskId": "1",
        "status": "in_progress"
      }
    }
  ]
}
```

这个例子展示了 todolist 不是由 runtime 主动推导出来的，而是由模型在看到工具定义和任务管理提示后主动生成工具调用，再由 runtime 可信执行和持久化。
