# 06 MCPTool 代理与命名前缀

> `tools/MCPTool/` 1086 行——把 MCP 工具变成 Claude Code 的 `Tool` 对象、和权限系统对齐、按 server-known patterns 分类 collapse 显示。本篇拆 MCPTool 基类设计、命名规范、`classifyForCollapse.ts` 600 行的服务名白名单。

## 6.1 MCPTool 是个"基类壳"

```ts
// tools/MCPTool/MCPTool.ts:27-77
export const MCPTool = buildTool({
  isMcp: true,
  isOpenWorld() { return false },
  name: 'mcp',
  maxResultSizeChars: 100_000,
  async description() { return DESCRIPTION },
  async prompt() { return PROMPT },
  get inputSchema() { return inputSchema() },
  get outputSchema() { return outputSchema() },
  async call() { return { data: '' } },
  async checkPermissions() {
    return {
      behavior: 'passthrough',
      message: 'MCPTool requires permission.',
    }
  },
  renderToolUseMessage,
  userFacingName: () => 'mcp',
  renderToolUseProgressMessage,
  renderToolResultMessage,
  isResultTruncated(output: Output): boolean {
    return isOutputLineTruncated(output)
  },
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return { tool_use_id: toolUseID, type: 'tool_result', content }
  },
} satisfies ToolDef<InputSchema, Output>)
```

77 行的"模板对象"——大部分方法**会被 client.ts:1769 的 `{ ...MCPTool, ...覆盖 }` 重写**：

```ts
// 来自 client.ts:1769:
return {
  ...MCPTool,
  name: skipPrefix ? tool.name : fullyQualifiedName,
  // ... 大量重写方法 ...
}
```

注释里 `// Overridden in mcpClient.ts` 标了 6 个被重写的字段：

- `name`
- `description()`
- `prompt()`
- `call()`
- `userFacingName()`
- `isOpenWorld()`（被 server's annotations 重写）

MCPTool 提供**模板默认值** + **共享方法**（renderToolUseMessage / mapToolResultToToolResultBlockParam / isResultTruncated）—— client.ts 在 fetchToolsForClient 里基于它**生成具体 server 的 tool 对象**。

## 6.2 inputSchema 是 `passthrough()`

```ts
export const inputSchema = lazySchema(() => z.object({}).passthrough())
```

`z.object({}).passthrough()` 让任何 input 都通过——**MCP server 定义自己的 schema**，Claude Code 不预先校验。

为什么不校验？因为 `inputSchema` 在 Tool 定义里有，但**实际 schema 由 MCP server 提供**（`tool.inputSchema`）—— client.ts:1813 把它注入到生成的 Tool 对象里：

```ts
inputJSONSchema: tool.inputSchema as Tool['inputJSONSchema'],
```

LLM 调用时**用 server 的 schema**校验。MCPTool 模板的 `inputSchema` 只是接口约束，不参与实际 validation。

## 6.3 outputSchema 是 `z.string()`

```ts
export const outputSchema = lazySchema(() =>
  z.string().describe('MCP tool execution result'),
)
```

输出类型固定 `string`——MCP tool 返回的所有内容**最终都变成字符串**给模型看。

实际 MCP 返回可能是 array of content blocks（text/image/etc）—— client.ts 的 `call()` 处理后转字符串。

## 6.4 maxResultSizeChars = 100KB

```ts
maxResultSizeChars: 100_000,
```

MCP 工具结果**最多 100KB**——超过会触发 truncation（写盘 + 给模型看 preview）。

对比其它工具：

| 工具 | maxResultSizeChars |
|------|-------------------|
| BashTool | 30K |
| MCPTool | 100K |
| FileRead | (无上限，走 lines + tokens) |

**MCP 更宽松**——某些 MCP 工具（数据库查询、API 返回 JSON）合理返回量比 bash 输出大。

`isResultTruncated` 用 `isOutputLineTruncated`——和 Bash 共享判定逻辑（看输出是不是被截断的行）。

## 6.5 checkPermissions 默认 passthrough

```ts
async checkPermissions(): Promise<PermissionResult> {
  return {
    behavior: 'passthrough',
    message: 'MCPTool requires permission.',
  }
}
```

**所有 MCP 工具默认走 user prompt**——`'passthrough'` 让权限决策跳到下一层。

但 client.ts:1814 的**重写版本**提供了 suggestion：

```ts
async checkPermissions() {
  return {
    behavior: 'passthrough',
    message: 'MCPTool requires permission.',
    suggestions: [
      {
        type: 'addRules',
        rules: [{ toolName: fullyQualifiedName, ruleContent: undefined }],
        behavior: 'allow',
        destination: 'localSettings',
      },
    ],
  }
}
```

用户点确认时可选 "always allow"——自动加 rule 到 settings。

## 6.6 mcp__server__tool 命名构建

`buildMcpToolName(serverName, toolName)` 实现（推断）：

```ts
function buildMcpToolName(serverName: string, toolName: string): string {
  const normalizedServer = normalizeNameForMCP(serverName)
  const normalizedTool = normalizeNameForMCP(toolName)
  return `mcp__${normalizedServer}__${normalizedTool}`
}
```

格式：`mcp__<server>__<tool>`。例：

| Server name | Tool name | 最终名 |
|-------------|-----------|--------|
| `github` | `create_issue` | `mcp__github__create_issue` |
| `supabase` | `query` | `mcp__supabase__query` |
| `claude.ai GitHub` | `list_repos` | `mcp__claude_ai_GitHub__list_repos` |
| `@anthropic/server` | `do_thing` | `mcp___anthropic_server__do_thing` |

注意 4 点：

### 1. `mcp__` 前缀

固定字面值——让权限系统能用通配符 `mcp__github__*` 匹配某 server 所有工具。

### 2. `__` 双下划线分隔

避免和工具名内的单下划线混淆。`create_issue` 内有 `_`，但 `__` 不会出现在合法工具名里。

### 3. normalize 规则

`normalization.ts:23`：

```ts
export function normalizeNameForMCP(name: string): string {
  return name.replace(/[^a-zA-Z0-9_]/g, '_')
}
```

非字母数字下划线 → 替换 `_`。让任何 server / tool 名都能进入 JavaScript identifier。

### 4. 长度限制

虽然没显式 length check，但 LLM 的 tool name slot 有上限（通常 64 字符）—— Claude.ai server 名带 `claude_ai_` 前缀可能撞上限。

## 6.7 与权限系统对齐

`mcp__server__tool` 命名直接服务权限规则：

```json
// ~/.claude/settings.json:
{
  "permissions": {
    "rules": [
      { "behavior": "allow", "toolName": "mcp__github__*" },
      { "behavior": "allow", "toolName": "mcp__supabase__query" },
      { "behavior": "deny", "toolName": "mcp__filesystem__write_file" }
    ]
  }
}
```

3 种模式：

- `mcp__github__*`：通配 github server 所有工具；
- `mcp__supabase__query`：精确单工具；
- `deny` 拒绝单工具。

[bashtool 04 permission 决策](../bashtool/04-permission决策核心.md) 讲的 wildcard 匹配机制——MCP 工具命名让这套规则直接复用。

## 6.8 SDK MCP no-prefix 例外

回顾 [05 client 核心](./05-client核心-连接与capability.md) 提过：

```ts
const skipPrefix =
  client.config.type === 'sdk' &&
  isEnvTruthy(process.env.CLAUDE_AGENT_SDK_MCP_NO_PREFIX)

return {
  ...MCPTool,
  name: skipPrefix ? tool.name : fullyQualifiedName,
  mcpInfo: { serverName: client.name, toolName: tool.name },
  // ...
}
```

skip-prefix 模式下：
- `name`: 直接用原 tool name（如 `Read`）；
- `mcpInfo` 仍然记录 server 名——**权限系统通过 mcpInfo 判定**而不是 name；
- 让 SDK MCP 能覆盖 builtin（如自定义 `Read` 替代内置）。

这种 **name vs mcpInfo 解耦**让命名规则和权限规则分离——name 给模型看、mcpInfo 给系统用。

## 6.9 classifyForCollapse.ts —— 600 行的服务名白名单

`classifyForCollapse.ts` 604 行——按 server name + tool name **分类 MCP 工具**为 search/read/list/write/destructive。

### 为什么需要分类

Claude Code UI 把"只读"工具调用折叠成单行（如 `[grep file.ts] (collapsed)`）—— 减少噪声。普通 BashTool 通过 `isReadOnlyBashCommand` 判定（`utils/bash/...`），FileTool 自己声明 `isReadOnly()`。

MCP 工具没有这种内置判定——`annotations.readOnlyHint` 是 hint 不是 ground truth。**Claude Code 维护一份针对常见 MCP server 的白名单**。

### SEARCH_TOOLS 白名单

`classifyForCollapse.ts:14-139` 是个 ~120 entry 的 Set：

```ts
const SEARCH_TOOLS = new Set([
  // Slack
  'slack_search_public',
  'slack_search_public_and_private',
  'slack_search_channels',
  // GitHub
  'search_code',
  'search_repositories',
  // Linear, Datadog, Sentry, Notion, Gmail, Google Drive, Jira, Atlassian,
  // Asana, Filesystem, Memory, Brave, Grafana, Stripe, PubMed, Firecrawl,
  // Exa, Perplexity, Tavily, Obsidian, MongoDB, Neo4j, Airtable, Todoist,
  // AWS, Terraform...
  // (覆盖 25+ 主流 MCP server)
])
```

注释（line 7-11）：

> Uses explicit per-tool allowlists for the most common MCP servers. Tool names are stable across installs (even when the server name varies, e.g., "slack" vs "claude_ai_Slack"), so matching is keyed on the tool name alone after normalizing camelCase/kebab-case to snake_case. Unknown tool names don't collapse (conservative).

3 个设计点：

### 1. 按 tool name 而不是 server name 匹配

server 名可能变（用户随便起 / Claude.ai 加前缀），**tool 名稳定**（MCP server 作者命名）。`slack_search_public` 在 slack server 和 claude_ai_Slack server 都叫一样。

### 2. 25+ 服务的常见 search/read tool 全列出

不是泛化（"含 search 字样的都算"）—— **精确白名单**避免误判。例如某个未知 server 有 `delete_search_history` 工具，泛化会误判为 search。

### 3. 未知工具保守不 collapse

```ts
// 未知 tool name 默认 false
```

如果工具名不在白名单——**不 collapse**（显示完整输出）。**保守优于聪明**——多展示信息比误隐藏好。

### 25+ 个 server 列表

注释里看到的 MCP server：

- **沟通**：Slack / Gmail
- **代码托管**：GitHub / GitLab
- **任务管理**：Linear / Jira / Asana / Todoist / Notion / Confluence
- **监控**：Datadog / Sentry / Grafana / PagerDuty
- **存储/数据库**：Supabase / MongoDB / Neo4j / Airtable / Filesystem / Memory
- **搜索**：Brave / Exa / Perplexity / Tavily / Firecrawl
- **AI 服务**：Claude.ai 多个 / Google Drive / Calendar
- **支付**：Stripe
- **研究**：PubMed
- **DevOps**：AWS / Terraform
- **笔记**：Obsidian
- **Git**：mcp-server-git

**覆盖主流 MCP server**——是个对真实生态的精细认知。

### READ_TOOLS / LIST_TOOLS 类似

`classifyForCollapse.ts:142+` 是 `READ_TOOLS`、再后面是 `LIST_TOOLS`——按相同模式列举。

最终：

```ts
export function classifyMcpToolForCollapse(serverName, toolName): { isSearch, isRead, isList } {
  const normalized = normalizeToSnakeCase(toolName)
  return {
    isSearch: SEARCH_TOOLS.has(normalized),
    isRead: READ_TOOLS.has(normalized),
    isList: LIST_TOOLS.has(normalized),
  }
}
```

调用方：client.ts:1810 的 `isSearchOrReadCommand`。

## 6.10 prompt.ts —— 仅 3 行

```ts
// tools/MCPTool/prompt.ts (全文 3 行)
export const PROMPT = ''
export const DESCRIPTION = 'MCP Tool'
```

**几乎是空的**——因为实际 prompt 来自 MCP server 的 `tool.description`（[05 client 核心](./05-client核心-连接与capability.md) 提过：`async prompt() { return tool.description ?? '' }`）。

MCPTool 自己的默认值没用——server 总会提供。

## 6.11 UI.tsx —— 渲染逻辑

`tools/MCPTool/UI.tsx` 402 行——MCP 工具调用 / 结果的 React 渲染。

| 函数 | 用途 |
|------|------|
| `renderToolUseMessage(input, ...)` | 调用前显示 "Calling mcp__github__create_issue with..." |
| `renderToolUseProgressMessage(progress)` | 实时显示进度（"Connecting..." / "Executing..."） |
| `renderToolResultMessage(output, isCollapsed)` | 显示结果（含 collapse 折叠） |

renderToolResultMessage 大致逻辑：

```ts
// 推断
if (isCollapsed && isSearchOrRead) {
  return <CollapsedRow>📦 {serverName}:{toolName} returned {N lines}</CollapsedRow>
}
return <FullOutput>{output}</FullOutput>
```

依赖 classifyForCollapse 的判定——只读工具 collapse 显示。

## 6.12 几个隐性设计判断

### 1. MCPTool 是模板而不是实例

77 行模板被 `{ ...MCPTool, override }` 用于生成每个 server 的具体 tool。**复用共享方法**避免每个 server tool 重复实现。

### 2. inputSchema passthrough

LLM 看到 server-provided schema、Claude Code 不做预校验。**信任 server 的 schema 定义**——MCP 协议就是这样设计的。

### 3. outputSchema 强制 string

MCP 返回可能是结构化（content array）—— Claude Code 转字符串简化。模型只能读字符串内容。

### 4. maxResultSizeChars 100KB

比 BashTool (30KB) 大——MCP 数据库 / API 工具合理返回量更大。

### 5. checkPermissions 默认 passthrough

让用户决定——但提供 suggestion 让用户能一键 "always allow"。**安全默认 + 用户便利**。

### 6. mcp__ 双下划线分隔

不会和工具名内的单下划线冲突——格式规整。

### 7. SDK no-prefix 用 mcpInfo 分离

name 给模型 / mcpInfo 给权限系统——**两种身份解耦**让 SDK MCP 能覆盖 builtin。

### 8. classifyForCollapse 用 tool name 匹配

tool 名稳定（server 作者命名）、server 名易变——按 tool 名匹配 robust。

### 9. SEARCH/READ/LIST 白名单 25+ 服务

不泛化（"含 search 字样"），**精确白名单**——避免 `delete_search_history` 这种误判。

### 10. 未知工具保守不 collapse

show 完整输出—— **保守优于聪明**。如果哪天加了新 MCP server，未识别工具默认显示全量信息。

### 11. prompt.ts 几乎空

因为实际 prompt 来自 server——MCPTool 自身没有意义的描述。

## 6.13 与其它专题串联

- **[05 client 核心](./05-client核心-连接与capability.md)**：MCPTool 被 fetchToolsForClient 用 `{ ...MCPTool, override }` 模板生成 server tools；
- **[core-models 02 Tool](../core-models/02-Tool-工具的统一抽象.md)**：MCPTool 是 Tool 接口的具体实现；
- **[bashtool 04 permission](../bashtool/04-permission决策核心.md)**：MCP 工具用 `mcp__server__tool` 命名让 wildcard 权限规则能用；
- **[messages-pipeline 02](../messages-pipeline/02-流式事件与assistant增量构造.md)**：MCP tool call 的 onProgress 走 createProgressMessage。

## 6.14 小结

- MCPTool 77 行——基类模板，方法在 client.ts:1769 用 `{ ...MCPTool, override }` 重写；
- inputSchema `passthrough()` —— MCP server 自己定义 schema，Claude Code 不预校验；
- outputSchema 强制 string —— 简化模型读取（实际可能 content array，转字符串）；
- `maxResultSizeChars: 100K` —— 比 BashTool (30K) 宽松，符合 MCP 数据库/API 场景；
- `checkPermissions` 默认 passthrough + 建议 always-allow 规则；
- `mcp__server__tool` 双下划线分隔——避免和工具名内 `_` 冲突；
- 命名规则与权限规则对齐——`mcp__github__*` 通配支持；
- SDK no-prefix 模式：name 给模型 / mcpInfo 给权限系统——双身份解耦；
- `classifyForCollapse.ts` 604 行白名单——按 tool name（稳定）而不是 server name（易变）匹配；
- SEARCH/READ/LIST 三类各 ~100+ entry，覆盖 25+ 主流 MCP server；
- 未知 tool 保守不 collapse——显示完整输出而不是误隐藏；
- `prompt.ts` 几乎空——实际 prompt 来自 server description。

下一篇 → [07 OAuth 与 3 套认证体系](./07-OAuth与3套认证体系.md)
