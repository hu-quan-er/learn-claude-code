# 05 client 核心：连接与 capability

> `client.ts` 3348 行——MCP 集成的"中枢"。本篇拆 `connectToServer` 入口、`fetchToolsForClient` / `fetchResources` / `fetchCommands` 的 LRU 缓存、能力协商、Unicode 清洗、`mcp__server__tool` 命名构建、tool call 的 url elicitation retry。

## 5.1 client.ts 的分区

3348 行大致 5 个区段：

```
1-200    imports + McpAuthError / McpToolCallError 类
200-300  utils helpers (URL 处理 / cache key)
300-500  fetch wrappers (createClaudeAiProxyFetch / wrapFetchWithTimeout)
500-1700 connectToServer (3348 行里最大的函数, ~1100 行)
1700-2000 areMcpConfigsEqual / fetchToolsForClient
2000-2400 fetchResourcesForClient / fetchCommandsForClient / prefetchAllMcpResources
2400-3348 callMCPToolWithUrlElicitationRetry / 工具调用 / 结果转换
```

最大的两个函数：

- `connectToServer` ~1100 行 (line 595-1700) —— 7 种 transport 分支
- `fetchToolsForClient` ~250 行 (line 1743-2000) —— tools/list 处理 + Tool 对象构造

## 5.2 connectToServer —— memoize 缓存的连接入口

```ts
// client.ts:595
export const connectToServer = memoize(
  async (serverRef: ScopedMcpServerConfig, name: string): Promise<MCPServerConnection> => {
    // 1. 计算 cache key
    // 2. 创建 transport (7 种分支)
    // 3. 创建 Client 实例
    // 4. 注册 ListRoots / elicitation handler
    // 5. race 连接 promise 与 timeout
    // 6. 拿 capabilities
    // 7. 错误归因 + 转 MCPServerConnection 状态
    // 8. cleanup callback
  },
  getServerCacheKey,
)
```

整个函数被 **memoize**——同 server config 多次调用返回同一个 promise / 同一个连接。

### getServerCacheKey

`client.ts:581-593`：

```ts
export function getServerCacheKey(
  serverRef: ScopedMcpServerConfig,
  name: string,
): string {
  return `${name}:${jsonStringify(serverRef)}`
}
```

key = `"${name}:${stringify(config)}"`——name + 完整 config 序列化。

含义：

- 同 name + 同 config → 复用连接；
- name 不变但 config 改了（如换 URL） → 不同 key → 新连接；
- 复用 server 内存 + 避免重复连接开销。

### 连接管理器调用

[10 连接管理](./10-连接管理与生命周期.md) 的 `useManageMCPConnections` 调 `connectToServer`——memoize 保证不重复连接同一 server。

## 5.3 McpAuthError 与错误类层级

`client.ts:152+`：

```ts
export class McpAuthError extends Error {
  constructor(public serverName: string, message?: string) {
    super(message)
    this.name = 'McpAuthError'
  }
}

export class McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
  extends TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
{
  // ...
}
```

两个专用错误类：

| 类 | 用途 |
|----|------|
| `McpAuthError` | OAuth / 认证失败 |
| `McpToolCallError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` | 工具调用失败 |

`TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 是个**特殊基类**——continue（变量名超长）表示"上报到 analytics 时已确认 error message 不含 PII"。

注释 inline 在变量名里——这种**长 marker 名字**避免开发者随便用这些类做不安全的 logging。

### isMcpSessionExpiredError

`client.ts:193+`：

```ts
export function isMcpSessionExpiredError(error: Error): boolean {
  // 识别 server 返回 "session expired" 类错误
  // 用于触发自动 reconnect
}
```

特定错误识别——某些 server 用 "session expired" 错误暗示 client 应该重连。这个 helper 让上层能自动处理。

## 5.4 5 个 server 状态

回顾 [00 总览](./00-总览与代码地图.md) 提过 `MCPServerConnection` 是 union type：

```ts
export type MCPServerConnection =
  | ConnectedMCPServer        // 连上了
  | FailedMCPServer           // 失败
  | NeedsAuthMCPServer        // 需要认证
  | PendingMCPServer          // 连接中
  | DisabledMCPServer         // 禁用
```

`connectToServer` 根据连接结果返回不同 type。状态决定 UI 显示和下游行为：

| 状态 | UI 显示 | fetchTools 行为 |
|------|---------|-----------------|
| connected | 绿色 ✓ | 返回工具列表 |
| failed | 红色 × | 返回 [] |
| needs-auth | 黄色提示 + Connect 按钮 | 返回 [] |
| pending | spinner | 返回 [] |
| disabled | 灰色 | 返回 [] |

`fetchToolsForClient` 第一行 (`client.ts:1745`)：

```ts
if (client.type !== 'connected') return []
```

**只有 connected 状态才尝试 fetch**——其它直接 return []。

## 5.5 fetchToolsForClient —— 250 行的 Tool 对象构造

`client.ts:1743+` 是把 MCP server 的 tool list 转成 Claude Code `Tool[]` 的关键函数。

### LRU 缓存 + key

```ts
export const fetchToolsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Tool[]> => { ... },
  // 第二参 cache size / key generator
)

const MCP_FETCH_CACHE_SIZE = 20
```

LRU 缓存 + 大小 20——避免无限增长。Cache key 基于 client name（稳定跨重连）。

为什么需要缓存？Tool 对象构造涉及 closure capture + schema parsing——重复构造浪费。同 server 重连后名字一样、tools 也一样 → cache hit。

### Step 1: 检查能力

```ts
async (client: MCPServerConnection): Promise<Tool[]> => {
  if (client.type !== 'connected') return []

  try {
    if (!client.capabilities?.tools) {
      return []
    }
    // ...
```

**能力协商**：server 在 init 时告诉 client 自己提供哪些能力（`capabilities`）。如果 server 没声明 `tools` 能力 → return [] 不查询。

这避免对**只提供 prompts/resources 的 server** 浪费 `tools/list` 调用。

### Step 2: 调用 tools/list

```ts
const result = (await client.client.request(
  { method: 'tools/list' },
  ListToolsResultSchema,
)) as ListToolsResult
```

发 JSON-RPC 请求 `tools/list`——server 返回 `{tools: [{name, description, inputSchema, ...}]}`。

### Step 3: Unicode 清洗

```ts
// Sanitize tool data from MCP server
const toolsToProcess = recursivelySanitizeUnicode(result.tools)
```

整个 tools 数组**走 Unicode sanitization**——这就是 [prompt-injection 03 Unicode 清洗](../prompt-injection/03-Unicode清洗管线.md) 讲过的入口之一。

为什么 MCP server 返回的工具描述需要清洗？因为：

- MCP server 是**外部代码**，可能包含 tag chars 攻击的 description；
- description 会进 LLM 上下文（作为 tool prompt）；
- 不清洗 → tag-chars attack 可能渗透到模型。

`recursivelySanitizeUnicode` 递归清洗对象所有字段 + 数组所有元素——一次性净化整个工具描述结构。

### Step 4: SDK MCP no-prefix 模式

```ts
const skipPrefix =
  client.config.type === 'sdk' &&
  isEnvTruthy(process.env.CLAUDE_AGENT_SDK_MCP_NO_PREFIX)
```

**双重 gate**：

- SDK type；
- 且环境变量 `CLAUDE_AGENT_SDK_MCP_NO_PREFIX` 设了。

满足时**跳过 `mcp__server__` 前缀**——让 SDK MCP 工具能用**原始名**覆盖 builtin。

用例：SDK 调用方想用 `Read` 名字提供自己的 FileRead 实现——覆盖 Claude Code 内置的。如果走 `mcp__sdk__Read` 前缀，模型可能同时看到 builtin `Read` 和 `mcp__sdk__Read` → 混淆。skip-prefix 让 SDK 的 Read **完全替代** builtin。

注释（`client.ts:1771-1773`）：

> In skip-prefix mode, use the original name for model invocation so MCP tools can override builtins by name. mcpInfo is used for permission checking.

**对模型**：看起来就是 `Read`。**对权限系统**：通过 `mcpInfo` 字段知道实际来自 MCP——能用 MCP 专属权限规则。

### Step 5: 构造 Tool 对象

```ts
return toolsToProcess.map((tool): Tool => {
  const fullyQualifiedName = buildMcpToolName(client.name, tool.name)
  return {
    ...MCPTool,  // 复用 MCPTool 基础实现
    name: skipPrefix ? tool.name : fullyQualifiedName,
    mcpInfo: { serverName: client.name, toolName: tool.name },
    isMcp: true,
    searchHint: tool._meta?.['anthropic/searchHint'] ? ... : undefined,
    alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,

    async description() { return tool.description ?? '' },
    async prompt() {
      const desc = tool.description ?? ''
      return desc.length > MAX_MCP_DESCRIPTION_LENGTH
        ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '… [truncated]'
        : desc
    },

    isConcurrencySafe() { return tool.annotations?.readOnlyHint ?? false },
    isReadOnly() { return tool.annotations?.readOnlyHint ?? false },
    isDestructive() { return tool.annotations?.destructiveHint ?? false },
    isOpenWorld() { return tool.annotations?.openWorldHint ?? false },
    isSearchOrReadCommand() { return classifyMcpToolForCollapse(client.name, tool.name) },
    toAutoClassifierInput(input) { return mcpToolInputToAutoClassifierInput(input, tool.name) },

    inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],

    async checkPermissions() { /* 见下文 */ },
    async call(args, context, _canUseTool, parentMessage, onProgress) { /* 见下文 */ },
  }
})
```

每个 MCP tool 变成完整的 `Tool` 对象——和 Claude Code 内置 Tool 接口完全一致。

### `tool._meta` 协议扩展

```ts
searchHint: typeof tool._meta?.['anthropic/searchHint'] === 'string'
  ? tool._meta['anthropic/searchHint'].replace(/\s+/g, ' ').trim() || undefined
  : undefined,
alwaysLoad: tool._meta?.['anthropic/alwaysLoad'] === true,
```

`_meta` 是 MCP 协议留给客户端/服务端约定的扩展字段——Claude Code 用 `anthropic/` 前缀的 key 加自己特定的 hint：

- `anthropic/searchHint`：搜索时显示的 hint 文本；
- `anthropic/alwaysLoad`：tool 是否始终加载到 schema（不走 dynamic discovery）。

注释（`client.ts:1776-1778`）：

> Collapse whitespace: _meta is open to external MCP servers, and a newline here would inject orphan lines into the deferred-tool list (formatDeferredToolLine joins on '\n').

**防注入**——server 可能在 `_meta` 里塞换行符。Claude Code 强制把空白 collapse 成单空格。

### `annotations` 协议字段

```ts
isConcurrencySafe() { return tool.annotations?.readOnlyHint ?? false },
isReadOnly() { return tool.annotations?.readOnlyHint ?? false },
isDestructive() { return tool.annotations?.destructiveHint ?? false },
isOpenWorld() { return tool.annotations?.openWorldHint ?? false },
```

MCP 协议的 **`annotations`** 是标准字段（不是 `_meta`）：

- `readOnlyHint`: 工具只读？
- `destructiveHint`: 工具可能破坏数据？
- `openWorldHint`: 工具访问外部资源（网络等）？

Claude Code 直接用这些 hint 决定**并发安全性** / 只读判定 / 危险标记 / open-world 标记。**server 自描述**——Claude Code 信任 server 告诉的属性。

### description 截断

```ts
async prompt() {
  const desc = tool.description ?? ''
  return desc.length > MAX_MCP_DESCRIPTION_LENGTH
    ? desc.slice(0, MAX_MCP_DESCRIPTION_LENGTH) + '… [truncated]'
    : desc
},
```

工具描述长度上限——`MAX_MCP_DESCRIPTION_LENGTH`（推断 ~2-4k 字符）。**防恶意 server 发巨长 description 撑爆 prompt**。

截断后加 `…[truncated]` 让模型知道有截断。

### checkPermissions —— 总是 passthrough + 建议规则

```ts
async checkPermissions() {
  return {
    behavior: 'passthrough' as const,
    message: 'MCPTool requires permission.',
    suggestions: [
      {
        type: 'addRules' as const,
        rules: [{
          toolName: fullyQualifiedName,
          ruleContent: undefined,
        }],
        behavior: 'allow' as const,
        destination: 'localSettings' as const,
      },
    ],
  }
}
```

**所有 MCP 工具默认走 ask user** —— `behavior: 'passthrough'` 让权限决策跳到下一层（user prompt）。

`suggestions` 提供一个**"always allow"快捷规则**——用户点确认时可以选"始终允许这个工具"，Claude Code 自动加 rule 到 localSettings：

```
Allow: mcp__github__create_issue
```

下次再调时不弹窗——直接放行。

## 5.6 call() —— MCP 工具调用主流程

`client.ts:1833+` 的 `call` 方法：

```ts
async call(args, context, _canUseTool, parentMessage, onProgress?) {
  const toolUseId = extractToolUseId(parentMessage)
  const meta = toolUseId ? { 'claudecode/toolUseId': toolUseId } : {}

  // 1. emit progress 'started'
  if (onProgress && toolUseId) {
    onProgress({
      toolUseID: toolUseId,
      data: {
        type: 'mcp_progress',
        status: 'started',
        serverName: client.name,
        toolName: tool.name,
      },
    })
  }

  const startTime = Date.now()
  const MAX_SESSION_RETRIES = 1

  // 2. retry loop (max 2 attempts)
  for (let attempt = 0; ; attempt++) {
    try {
      const connectedClient = await ensureConnectedClient(client)
      const mcpResult = await callMCPToolWithUrlElicitationRetry({
        client: connectedClient,
        clientConnection: client,
        tool: tool.name,
        args,
        meta,
        signal: context.abortController.signal,
        setAppState: context.setAppState,
        onProgress: /* ... */,
        handleElicitation: context.handleElicitation,
      })

      // 3. emit progress 'completed'
      if (onProgress && toolUseId) {
        onProgress({
          toolUseID: toolUseId,
          data: {
            type: 'mcp_progress',
            status: 'completed',
            serverName: client.name,
            toolName: tool.name,
            elapsedTimeMs: Date.now() - startTime,
          },
        })
      }

      // 4. 返回结果
      return {
        data: mcpResult.content,
        ...((mcpResult._meta || mcpResult.structuredContent) && { /* ... */ }),
      }
    } catch (error) {
      // session expired retry
      if (isMcpSessionExpiredError(error) && attempt < MAX_SESSION_RETRIES) {
        // 触发 reconnect 再重试
        continue
      }
      throw error
    }
  }
}
```

### session expired retry

`isMcpSessionExpiredError` 识别 server 主动断开 session 的错误——transport 自动 reconnect，重新调用工具。最多 retry 1 次（避免循环）。

这是给 **server 主动让 client 重连** 的场景（如 server 定期 rotate session token）。

### 进度上报

`onProgress` 在 started/completed/failed 都 emit 事件——UI 能显示 spinner 和耗时。

### `meta: 'claudecode/toolUseId'`

```ts
const meta = toolUseId ? { 'claudecode/toolUseId': toolUseId } : {}
```

Claude Code 在 MCP 请求的 `_meta` 里塞 `claudecode/toolUseId`——MCP server 可以用这个 ID 关联日志、做去重等。

`claudecode/` 是 Claude Code 自己的 namespace（类似 `anthropic/`）。

## 5.7 fetchResourcesForClient —— 资源列表

```ts
// client.ts:2000
export const fetchResourcesForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<ServerResource[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.resources) return []

    const result = await client.client.request(
      { method: 'resources/list' },
      ListResourcesResultSchema,
    )

    return result.resources.map(r => ({ ...r, server: client.name }))
  },
  /* ... */
)
```

类似 fetchTools——能力检查 + JSON-RPC 请求 + 映射成 Claude Code 类型。

资源用 URL 寻址（`file://...` / `db://...`）—— Claude Code 把它们当 attachment 注入 prompt。

## 5.8 fetchCommandsForClient —— prompts

```ts
// client.ts:2033
export const fetchCommandsForClient = memoizeWithLRU(
  async (client: MCPServerConnection): Promise<Command[]> => {
    if (client.type !== 'connected') return []
    if (!client.capabilities?.prompts) return []

    const result = await client.client.request(
      { method: 'prompts/list' },
      ListPromptsResultSchema,
    )

    // Sanitize prompt data from MCP server
    const promptsToProcess = recursivelySanitizeUnicode(result.prompts)
    // ...
    // 转成 Claude Code Command (slash command 风格)
  }
)
```

MCP server 的 prompts 变成 Claude Code 的 **slash command** —— `/server:prompt-name` 形式。用户 `/github:summarize-issue 123` → 走 MCP server 拿 prompt 模板 → 注入对话。

这就是为什么 `Command` 类型有 `isMcp` 字段（[core-models 01 Command](../core-models/01-Command-命令的统一抽象.md) 讲过）。

## 5.9 prefetchAllMcpResources —— 启动时批量预取

`client.ts:2408+`：

```ts
export function prefetchAllMcpResources(
  clients: MCPServerConnection[],
): Promise<void> {
  return Promise.all(
    clients
      .filter(c => c.type === 'connected')
      .flatMap(client => [
        fetchToolsForClient(client),
        fetchResourcesForClient(client),
        fetchCommandsForClient(client),
      ])
  ).then(() => undefined)
}
```

启动时**所有 connected server** 并发预取 tools/resources/commands——填充 cache 让首次调用 instant。

如果不预取——首次 `tools/list` 在用户第一次启动时才发生 → 用户感觉"启动很快但首工具调用慢"。预取后体验一致。

并发 fetch——N 个 server × 3 个 endpoint 一起飞，比串行快 N×3 倍。

## 5.10 buildMcpToolName —— 命名格式

`buildMcpToolName(serverName, toolName)` 实现：

```ts
function buildMcpToolName(serverName: string, toolName: string): string {
  const normalizedServer = normalizeNameForMCP(serverName)
  const normalizedTool = normalizeNameForMCP(toolName)
  return `mcp__${normalizedServer}__${normalizedTool}`
}
```

格式：`mcp__<server>__<tool>`。例：

- `mcp__github__create_issue`
- `mcp__supabase__query_table`
- `mcp__claude_ai_GitHub__list_repos`（Claude.ai proxy server 名字有 `claude_ai_` 前缀）

normalize 规则（`normalization.ts:23`）—— 把 `-` 等非 identifier 字符变 `_`：

```
"create-issue" → "create_issue"
"my server" → "my_server"
"@anthropic/github" → "_anthropic_github"
```

JavaScript identifier 友好——能在 LLM 的 tool name slot 里直接用。

## 5.11 几个隐性设计判断

### 1. memoize connectToServer

避免重复连接同 server——cache key 含完整 config 让 config 改自动 reconnect。

### 2. LRU 缓存 fetchTools 限 20

server 数量上限 20——一般用户最多 5-10 个，缓存够用。

### 3. 能力协商 first，避免无效调用

`if (!client.capabilities?.tools) return []`——server 没声明能力就不调 list。**协议层的优化**。

### 4. Unicode 清洗在 server 返回路径

`recursivelySanitizeUnicode` 在 fetchTools 和 fetchCommands 都用——MCP server 是不可信外部源。Claude Code 信任协议但不信任 server 内容。

### 5. SDK MCP no-prefix 模式

双重 gate 防误用——`type === 'sdk'` + 环境变量。让 SDK 的 MCP 工具能覆盖 builtin 提供同名实现。

### 6. `_meta` vs `annotations` 双通道

- `annotations`: MCP 协议标准字段（readOnlyHint / destructiveHint 等）
- `_meta`: 协议预留扩展，Claude Code 用 `anthropic/` 前缀加自己的 hint

**协议标准 + 厂商扩展**双轨——既兼容协议又能加 Anthropic 特有 metadata。

### 7. description 截断 + 标注

防恶意 server 发巨长 description——截断并显示 `[truncated]` 让模型知道。

### 8. _meta 字段强制 collapse 空白

防 server 在 hint 里塞换行符注入到 deferred-tool list——一行一 tool 的列表会被换行符破坏。

### 9. checkPermissions 总是 passthrough + 建议规则

MCP 工具默认让用户决定——但提供 "always allow" 快捷规则建议。**安全默认 + 用户便利**。

### 10. session expired retry 一次

server 主动让 client 重连场景——自动 retry 一次，避免循环。

### 11. claudecode/toolUseId meta

Claude Code 在 MCP 请求里塞 toolUseId——让 server 能关联日志。**双向 metadata 通道**。

### 12. prefetchAll 启动时并发

避免首次工具调用慢——启动时全部预取。N × 3 个 endpoint 并发飞。

## 5.12 与已有专题的串联

- **[prompt-injection 03 Unicode 清洗](../prompt-injection/03-Unicode清洗管线.md)**：fetchTools / fetchCommands 调 `recursivelySanitizeUnicode` 防 tag-chars 攻击；
- **[core-models 02 Tool](../core-models/02-Tool-工具的统一抽象.md)**：fetchToolsForClient 把 MCP tool 转成 Claude Code `Tool` 对象；
- **[core-models 01 Command](../core-models/01-Command-命令的统一抽象.md)**：fetchCommandsForClient 把 MCP prompts 转成 Slash Command；
- **[messages-pipeline 02 流式](../messages-pipeline/02-流式事件与assistant增量构造.md)**：MCP tool call 的 progress 走 `createProgressMessage`；
- **[bashtool 04 permission](../bashtool/04-permission决策核心.md)**：MCP 的 `checkPermissions` 也走同样的 permission 系统，但默认 passthrough。

## 5.13 小结

- client.ts 3348 行核心：connectToServer ~1100 行 + fetchToolsForClient ~250 行；
- `connectToServer` memoize 缓存避免重复连接；
- 5 种 server 状态：connected / failed / needs-auth / pending / disabled；
- `fetchToolsForClient` LRU 缓存 20——同 server 重连后 cache hit；
- 能力协商先行——server 没声明能力就不调 list；
- 入口处 `recursivelySanitizeUnicode` —— [prompt-injection 03] 的 MCP 入口；
- SDK no-prefix 模式：`CLAUDE_AGENT_SDK_MCP_NO_PREFIX` 让 SDK MCP 工具覆盖 builtin；
- `_meta` 用 `anthropic/` 前缀添加 hint + 强制 collapse 空白防注入；
- `annotations` 是 MCP 协议字段（readOnlyHint / destructiveHint 等）—— Claude Code 信任 server 自描述；
- description 截断防长 prompt 攻击；
- `checkPermissions` 默认 passthrough + 建议 "always allow" 规则；
- `claudecode/toolUseId` meta 让 server 关联日志；
- `prefetchAllMcpResources` 启动时并发预取所有 server 的能力。

下一篇 → [06 MCPTool 代理与命名前缀](./06-MCPTool代理与命名前缀.md)
