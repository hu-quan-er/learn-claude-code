# 04 SDK 与 InProcess transport

> 3 种"非进程间"的 MCP transport——`InProcessTransport`（同进程通过 microtask 链）、`SdkControlTransport`（CLI ↔ SDK 跨进程通过 stdio 控制协议）、`claudeai-proxy`（Claude.ai 后端代理）。本篇拆它们如何绕开"传统 IPC"提供 MCP 协议体验。

## 4.1 为什么需要"非传统"transport

回顾 [02 stdio](./02-stdio与子进程transport.md) 和 [03 HTTP/SSE/WS](./03-HTTP-SSE-WebSocket-transport.md)——它们都是"经典 IPC"：

- stdio：进程间 stdin/stdout
- HTTP/SSE/WS：网络 socket

但 3 个场景**不适合**这些 transport：

### 场景 1：Claude Code 内置的 MCP server

`isClaudeInChromeMCPServer` / `isComputerUseMCPServer` 这些 server 实际**就在 Claude Code 代码库**——给它们 spawn 一个子进程（用 stdio）会浪费 325MB 内存。

**InProcessTransport** 解决——同进程内通过 microtask 调用，零 IPC 开销。

### 场景 2：SDK 调用方注入的 MCP server

Claude Code SDK 允许第三方程序**作为库**调用 Claude——这种场景下 SDK 进程是宿主，CLI 是它启动的子进程。SDK 程序可能想注入自己的 MCP server，但：

- 这个 server 在**SDK 进程**里跑，不在 CLI 进程；
- 不能用 stdio（CLI 用的 stdio 已经是和 SDK 进程通信的）；
- 不能用 HTTP（需要起服务器、防火墙等）。

**SdkControlTransport** 解决——复用 CLI ↔ SDK 已有的 stdio 控制通道，**把 MCP 消息包装成控制请求**。

### 场景 3：Claude.ai 用户的 MCP server

Claude.ai 网页用户的 MCP server 由 Anthropic **代理托管**：

- 用户在 Claude.ai 上配 server，token 在浏览器；
- Claude Code 客户端不直接连用户 server，而是**通过 Anthropic 代理**调用；
- 代理处理认证、统一身份、统一日志。

**claudeai-proxy** transport 解决——HTTP 但走 Anthropic 的代理 URL。

## 4.2 InProcessTransport —— 63 行的优雅设计

`InProcessTransport.ts` 全文 63 行——但实现了 MCP 协议**完整的内存级 IPC**。

### 完整实现

```ts
class InProcessTransport implements Transport {
  private peer: InProcessTransport | undefined
  private closed = false

  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void

  /** @internal */
  _setPeer(peer: InProcessTransport): void {
    this.peer = peer
  }

  async start(): Promise<void> {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.closed) {
      throw new Error('Transport is closed')
    }
    // Deliver to the other side asynchronously to avoid stack depth issues
    // with synchronous request/response cycles
    queueMicrotask(() => {
      this.peer?.onmessage?.(message)
    })
  }

  async close(): Promise<void> {
    if (this.closed) return
    this.closed = true
    this.onclose?.()
    if (this.peer && !this.peer.closed) {
      this.peer.closed = true
      this.peer.onclose?.()
    }
  }
}

export function createLinkedTransportPair(): [Transport, Transport] {
  const a = new InProcessTransport()
  const b = new InProcessTransport()
  a._setPeer(b)
  b._setPeer(a)
  return [a, b]
}
```

### 设计核心：linked pair

```ts
const [clientTransport, serverTransport] = createLinkedTransportPair()
await mcpServer.connect(serverTransport)
await mcpClient.connect(clientTransport)
```

**两个 transport 实例互相指向对方**——`a` 的 `send()` 通过 peer 调 `b.onmessage`。client 和 server 各 attach 一个。

### queueMicrotask —— 异步交付防栈溢出

```ts
queueMicrotask(() => {
  this.peer?.onmessage?.(message)
})
```

注释明确：

> Deliver to the other side asynchronously to avoid stack depth issues with synchronous request/response cycles

为什么需要异步？因为 **同步 request/response 链**会让调用栈无限增长：

```
client.send(req)
  → server.onmessage(req)  (同步)
    → server.handler(req)
      → server.send(resp)
        → client.onmessage(resp)  (同步)
          → client.handler(resp)
            → ... 下一个 req ...
              → ... 无限嵌套 ...
```

每个请求-响应都在栈上累积——足够深就 RangeError。

**`queueMicrotask`** 让 onmessage 在**下一个 microtask 里**调用——栈被清空。代价是延迟一个 microtask（< 1ms）。

### 对称关闭

```ts
async close(): Promise<void> {
  if (this.closed) return
  this.closed = true
  this.onclose?.()
  if (this.peer && !this.peer.closed) {
    this.peer.closed = true
    this.peer.onclose?.()
  }
}
```

任一端 close → **双向通知**。peer 也设 closed = true 防止双重关闭。

idempotent（`if (this.closed) return`）——可以 close 多次不报错。

### 为什么 start() 是空的

```ts
async start(): Promise<void> {}
```

MCP `Transport` 接口要求 start 方法——但 InProcess 不需要任何启动逻辑（peer 已经在构造时 link）。**留空满足接口**。

## 4.3 InProcessTransport 的实际用法

`client.ts:910-924`（Chrome MCP server 路径）：

```ts
} else if (
  (serverRef.type === 'stdio' || !serverRef.type) &&
  isClaudeInChromeMCPServer(name)
) {
  const { createChromeContext } = await import('../../utils/claudeInChrome/mcpServer.js')
  const { createClaudeForChromeMcpServer } = await import('@ant/claude-for-chrome-mcp')
  const { createLinkedTransportPair } = await import('./InProcessTransport.js')

  const context = createChromeContext(serverRef.env)
  inProcessServer = createClaudeForChromeMcpServer(context)
  const [clientTransport, serverTransport] = createLinkedTransportPair()
  await inProcessServer.connect(serverTransport)
  transport = clientTransport
  logMCPDebug(name, `In-process Chrome MCP server started`)
}
```

5 步：

1. **动态 import**：避免主路径加载（只在需要时）；
2. **创建 server**：`createClaudeForChromeMcpServer(context)`——Chrome MCP server 的实例；
3. **创建 transport pair**：两端互相 link；
4. **server 连 server-side transport**：server 准备好接收消息；
5. **transport = client-side**：等会被 MCP Client 用来连接。

后续 `client.connect(transport)` —— client 通过 client transport 发消息 → microtask → server transport.onmessage → server 处理 → server.send → microtask → client transport.onmessage。

整个流程**没有进程 / 网络 / 文件**——纯内存。

## 4.4 配置成 stdio 但走 InProcess —— 协议兼容性 trick

`isClaudeInChromeMCPServer(name)` 是个白名单检查——某些 server 名字被识别为"in-process 候选"。

```json
// 用户配置 (~/.claude.json):
{
  "mcpServers": {
    "claude-for-chrome": {
      "type": "stdio",
      "command": "claude-for-chrome",
      "args": []
    }
  }
}
```

用户配的是 `stdio`——但 client.ts 看到名字匹配 `isClaudeInChromeMCPServer` → **跳过 spawn**，直接用 InProcessTransport。

为什么这样设计？

- **协议兼容**：用户 / 文档 / migration 都把它当 stdio 看；
- **实现优化**：内部知道这个 server 在自己代码库里 → 走 in-process 省资源；
- **零迁移成本**：用户不需要改 type、Claude Code 升级后自动优化。

**配置不变、实现变**——经典的实现细节隐藏。

## 4.5 SdkControlTransport —— CLI ↔ SDK 双向

`SdkControlTransport.ts` 136 行实现 **2 个 Transport class**——一个在 CLI 端、一个在 SDK 端。它们通过**已有的 stdio 控制通道**通信。

### 架构

```
┌─────────────────────────────────────────────────────────────┐
│  SDK Process (宿主程序, 嵌入 Claude SDK)                     │
│                                                              │
│   ┌──────────────────┐         ┌──────────────────────┐     │
│   │ MCP Server       │ ←──────│ SdkControlServer     │     │
│   │ (用户写的代码)    │         │ Transport            │     │
│   └──────────────────┘         └──────────────────────┘     │
│                                            │                 │
│                                            │ via Query       │
│                                            │ structured IO   │
└────────────────────────────────────────────┼─────────────────┘
                                             │
                                       stdout/stdin
                                             │
┌────────────────────────────────────────────┼─────────────────┐
│  CLI Process (claude-code 进程)              │                 │
│                                            │                 │
│   ┌──────────────────────┐         ┌──────▼──────────┐     │
│   │ MCP Client           │ ←──────│ SdkControl       │     │
│   │ (Claude Code 内部)   │         │ ClientTransport  │     │
│   └──────────────────────┘         └──────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

注释（`SdkControlTransport.ts:3-37`）描述完整消息流：

> ### CLI → SDK (via SdkControlClientTransport)
> 1. CLI's MCP Client calls a tool → sends JSONRPC request to SdkControlClientTransport
> 2. Transport wraps the message in a control request with server_name and request_id
> 3. Control request is sent via stdout to the SDK process
> 4. SDK's StructuredIO receives the control response and routes it back to the transport
> 5. Transport unwraps the response and returns it to the MCP Client

### SdkControlClientTransport （CLI 端）

```ts
export class SdkControlClientTransport implements Transport {
  private isClosed = false

  onclose?: () => void
  onerror?: (error: Error) => void
  onmessage?: (message: JSONRPCMessage) => void

  constructor(
    private serverName: string,
    private sendMcpMessage: SendMcpMessageCallback,
  ) {}

  async start(): Promise<void> {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) throw new Error('Transport is closed')

    // Send the message and wait for the response
    const response = await this.sendMcpMessage(this.serverName, message)

    // Pass the response back to the MCP client
    if (this.onmessage) {
      this.onmessage(response)
    }
  }

  async close(): Promise<void> { /* ... */ }
}
```

关键设计：

### `serverName` 路由

```ts
constructor(
  private serverName: string,
  private sendMcpMessage: SendMcpMessageCallback,
) {}
```

`serverName` 让一个 stdio 控制通道支持**多个 SDK MCP server**——每个 server 一个 transport 实例，sendMcpMessage 时带 server name 让 SDK 路由到对应 server。

### `SendMcpMessageCallback` 类型

```ts
export type SendMcpMessageCallback = (
  serverName: string,
  message: JSONRPCMessage,
) => Promise<JSONRPCMessage>
```

**Promise-based callback**——sendMcpMessage 返回 Promise，resolve 时拿到 response。控制流是 request/response 同步配对。

### send-and-receive 一次性完成

```ts
async send(message: JSONRPCMessage): Promise<void> {
  const response = await this.sendMcpMessage(this.serverName, message)
  if (this.onmessage) {
    this.onmessage(response)
  }
}
```

注意：**send 内部直接 await response**——和典型 transport (`send` 只发不等)不同。

为什么？因为底层控制通道是 request/response 模型——每个发出去的 request 都对应一个 response。SdkControlTransport 把这种语义暴露给 MCP Client。

### SdkControlServerTransport （SDK 端）

```ts
export class SdkControlServerTransport implements Transport {
  constructor(private sendMcpMessage: (message: JSONRPCMessage) => void) {}

  async send(message: JSONRPCMessage): Promise<void> {
    if (this.isClosed) throw new Error('Transport is closed')
    // Simply pass the response back through the callback
    this.sendMcpMessage(message)
  }
}
```

更简单——SDK 端的 transport 只是把 server 的 response 通过 callback 传出去。注释（line 107）：

> Note: Query handles all request/response correlation and async flow.

**SDK 端不维护状态**——Query 模块（SDK 内部）追踪 pending request、调 `transport.onmessage`、收 server 响应、把响应送回 CLI。

### 不对称 Transport

CLI 端：维护 `isClosed` 状态 + 主动调 `sendMcpMessage` + 等 response
SDK 端：只 forward 消息，状态由 Query 维护

这种**不对称**反映两端职责不同——CLI 是 client（主动发请求等回复），SDK 是 server（被动收请求发响应）。

## 4.6 claudeai-proxy —— Anthropic 代理

`client.ts:868-904` 是 claudeai-proxy 处理：

```ts
} else if (serverRef.type === 'claudeai-proxy') {
  const tokens = getClaudeAIOAuthTokens()
  if (!tokens) throw new Error('No claude.ai OAuth token found')

  const oauthConfig = getOauthConfig()
  const proxyUrl = `${oauthConfig.MCP_PROXY_URL}${oauthConfig.MCP_PROXY_PATH.replace('{server_id}', serverRef.id)}`

  const fetchWithAuth = createClaudeAiProxyFetch(globalThis.fetch)

  const proxyOptions = getProxyFetchOptions()
  const transportOptions: StreamableHTTPClientTransportOptions = {
    fetch: wrapFetchWithTimeout(fetchWithAuth),
    requestInit: {
      ...proxyOptions,
      headers: {
        'User-Agent': getMCPUserAgent(),
        'X-Mcp-Client-Session-Id': getSessionId(),
      },
    },
  }

  transport = new StreamableHTTPClientTransport(new URL(proxyUrl), transportOptions)
}
```

### proxyUrl 构造

```ts
const proxyUrl = `${oauthConfig.MCP_PROXY_URL}${oauthConfig.MCP_PROXY_PATH.replace('{server_id}', serverRef.id)}`
```

URL 模板替换——例：

- `MCP_PROXY_URL = "https://api.anthropic.com"`
- `MCP_PROXY_PATH = "/api/mcp_servers/{server_id}/proxy"`
- `serverRef.id = "github-abc123"`
- → `https://api.anthropic.com/api/mcp_servers/github-abc123/proxy`

每个 server 一个独立 proxy endpoint。

### `createClaudeAiProxyFetch`

`auth.ts:372+`：

```ts
export function createClaudeAiProxyFetch(innerFetch: FetchLike): FetchLike {
  return async (url, init) => {
    const tokens = getClaudeAIOAuthTokens()
    if (!tokens) throw new Error('No claude.ai OAuth token')

    return innerFetch(url, {
      ...init,
      headers: {
        ...init?.headers,
        Authorization: `Bearer ${tokens.accessToken}`,
        // Plus 其它 Claude.ai 专用 headers
      },
    })
  }
}
```

**自动注入 Claude.ai OAuth token**——proxy 收到请求后验证 token、转发到实际 MCP server、把 response 传回。

### 走 Streamable HTTP transport

注意 claudeai-proxy **复用** `StreamableHTTPClientTransport`——不是新 transport 类型。只是配置不同（不同 URL、不同认证）。

这种**复用现有 transport class，定制配置**的模式——避免再写一个完整 transport class。

### 复杂度被代理隐藏

注释（隐含）：用户看到 server 是 `claudeai-proxy` type、id 是某个字符串——背后的细节（OAuth、proxy URL、headers）都被 transport 配置藏掉了。

模型也不感知——`mcp__claude_ai_GitHub__create_issue` 工具看起来和普通 MCP 工具一样。

## 4.7 claude.ai server 自动 fetch

`claudeai.ts:39+` 的 `fetchClaudeAIMcpConfigsIfEligible`：

```ts
export const fetchClaudeAIMcpConfigsIfEligible = memoize(
  async (): Promise<Record<string, ScopedMcpServerConfig>> => {
    if (isEnvDefinedFalsy(process.env.ENABLE_CLAUDEAI_MCP_SERVERS)) return {}

    const tokens = getClaudeAIOAuthTokens()
    if (!tokens?.accessToken) return {}

    // 检查 scope: 必须有 user:mcp_servers
    if (!tokens.scopes?.includes('user:mcp_servers')) return {}

    const baseUrl = getOauthConfig().BASE_API_URL
    const url = `${baseUrl}/v1/mcp_servers?limit=1000`

    const response = await axios.get<ClaudeAIMcpServersResponse>(url, {
      headers: {
        Authorization: `Bearer ${tokens.accessToken}`,
        'Content-Type': 'application/json',
        'anthropic-beta': MCP_SERVERS_BETA_HEADER,
        'anthropic-version': '2023-06-01',
      },
      timeout: 5000,
    })

    const configs: Record<string, ScopedMcpServerConfig> = {}
    const usedNormalizedNames = new Set<string>()

    for (const server of response.data.data) {
      const baseName = `claude.ai ${server.display_name}`
      // ... 处理名字冲突 ...

      configs[normalizedName] = {
        type: 'claudeai-proxy',
        url: server.url,
        id: server.id,
        scope: 'claudeai',
      }
    }

    return configs
  },
)
```

**启动时自动调用** Claude.ai API 拉用户配置的 server 列表：

- memoized——一次 session 只 fetch 一次；
- 5 秒超时——快速失败让 Claude Code 启动不卡；
- scope `user:mcp_servers` 必须存在——OAuth 时申请的权限；
- 名字冲突自动加 (2)/(3) 后缀；
- 配置打 `scope: 'claudeai'`（[01 配置](./01-McpServerConfig与8种transport-schema.md) 的 7 级 scope 之一）。

### `MCP_SERVERS_BETA_HEADER`

```ts
const MCP_SERVERS_BETA_HEADER = 'mcp-servers-2025-12-04'
```

每个 beta feature 一个版本字符串——让服务端能精确控制 client 兼容性。

### `user:mcp_servers` scope 检查

```ts
if (!tokens.scopes?.includes('user:mcp_servers')) return {}
```

OAuth token 必须有 `user:mcp_servers` scope——否则不 fetch。这避免没勾 MCP 权限的用户被自动加入 server。

### 注释里的 print mode 注意

```ts
// Check for user:mcp_servers scope directly instead of isClaudeAISubscriber().
// In non-interactive mode, isClaudeAISubscriber() returns false when ANTHROPIC_API_KEY
// is set (even with valid OAuth tokens) because preferThirdPartyAuthentication() causes
// isAnthropicAuthEnabled() to return false. Checking the scope directly allows users
// with both API keys and OAuth tokens to access claude.ai MCPs in print mode.
```

**真实 bug 修复**——非交互模式（CI / print mode）下用户可能同时有 API key 和 OAuth token。原来用 `isClaudeAISubscriber()` 判定 → API key 优先 → 即使 OAuth 有效也认为不订阅 → MCP server 不可用。

修复：**直接看 OAuth token scope**——绕过 subscriber 判定逻辑。

## 4.8 vscodeSdkMcp —— Claude Code 自己反向调 VSCode

`vscodeSdkMcp.ts` 112 行——一个**特殊的 in-process server**，用于 Claude Code 向 VSCode 发通知：

```ts
let vscodeMcpClient: ConnectedMCPServer | null = null

export function notifyVscodeFileUpdated(
  filePath: string,
  oldContent: string | null,
  newContent: string | null,
): void {
  if (process.env.USER_TYPE !== 'ant' || !vscodeMcpClient) return

  void vscodeMcpClient.client
    .notification({
      method: 'file_updated',
      params: {
        path: filePath,
        oldContent,
        newContent,
      },
    })
    .catch(/* ... */)
}
```

用途：**Claude Code 编辑文件后通知 VSCode 刷新**。

注意：

- 走 MCP 协议但用 `notification`（不期待 response）——fire-and-forget；
- gated `USER_TYPE === 'ant'`——只对内部用户启用；
- 全局单例（`vscodeMcpClient`）——VSCode 扩展启动 Claude Code 时建立连接。

这是 **Claude Code 既当 MCP client 又当 MCP "事件 emitter"** 的例子——把 MCP 协议反向用来发通知。

## 4.9 4 种 transport 对比

| transport | IPC 介质 | 用途 | 行数 |
|-----------|---------|------|-----|
| `stdio` (02) | 子进程 stdin/stdout | 本地命令 server | (SDK) |
| `http/sse/ws` (03) | 网络 socket | 远程 server | (SDK) |
| **`InProcessTransport`** | microtask | Claude Code 内置 server | 63 |
| **`SdkControlTransport` Client/Server** | CLI ↔ SDK 控制通道 | SDK 嵌入 | 136 |
| **`claudeai-proxy`** | HTTP (走 Anthropic 代理) | Claude.ai 用户 | 164 |

每种解决不同部署场景——**MCP 协议的传输层抽象在不同环境下展现的灵活性**。

## 4.10 几个隐性设计判断

### 1. InProcessTransport 用 queueMicrotask

避免栈溢出——同步 request/response 链在大量调用下会爆栈。**用 microtask 异步化**是 Node.js 标准模式。

### 2. linked pair 双向 close

idempotent + 双向通知——一端 close 另一端立刻知道。**对称设计**让两端代码相同。

### 3. 配置成 stdio 但实际 in-process

零迁移成本——用户配置不变。**实现细节隐藏**让 Claude Code 能优化某些 server 的实现而不破坏用户体验。

### 4. SdkControlTransport 不对称

CLI 端主动等 response、SDK 端被动 forward——反映两端职责。**接口对称但语义不同**。

### 5. serverName 路由

一个控制通道支持多 server——用 serverName 字段区分。**复用通道**省 IPC 设置。

### 6. claudeai-proxy 复用 StreamableHTTPClientTransport

不写新 transport 类——只配置 URL 和 fetch wrapper。**复用现有类**。

### 7. claude.ai server 自动 fetch + memoize

启动一次性拉所有 server——之后不重复。**用户零配置**——登录 Claude.ai 后 server 自动可用。

### 8. user:mcp_servers scope 检查

直接看 OAuth scope 而不是 isClaudeAISubscriber——修复非交互模式的真实 bug。注释引用具体场景。

### 9. memoize 5s timeout 快速失败

服务不可用时不卡启动——5 秒后 return {}。**快速降级**让 Claude Code 离线也能用。

### 10. vscodeSdkMcp 反向通知

Claude Code 当 MCP server 给 VSCode 发通知——**MCP 不止是工具调用**。

## 4.11 与 [swarm 04 In-process backend](../swarm/04-In-process-backend.md) 对比

| 维度 | swarm InProcessBackend | MCP InProcessTransport |
|------|----------------------|----------------------|
| 隔离机制 | AsyncLocalStorage | 内存 link |
| 通信介质 | 文件 mailbox | microtask |
| 抽象层级 | TeammateExecutor 接口 | Transport 接口 |
| 对称性 | 不对称（lead vs worker） | 对称（client/server peer） |
| 用途 | 多 teammate 协作 | server 调用工具 |

两个都叫"in-process"——但目的截然不同。swarm 是为多 actor，MCP 是为协议复用。

## 4.12 小结

- 3 种"非传统"transport 解决 stdio/HTTP/WS 不适合的场景；
- `InProcessTransport` 63 行优雅设计：linked pair、queueMicrotask 防栈溢出、对称 close；
- 配置成 stdio 但实际 in-process——零迁移成本的实现优化（Chrome / Computer Use server）；
- `SdkControlTransport` 不对称两端 Class——CLI 主动等 response、SDK 被动 forward；
- `serverName` 字段让一个控制通道路由多 SDK MCP server；
- `claudeai-proxy` 复用 `StreamableHTTPClientTransport`——只是配置不同 URL；
- Claude.ai server 启动时自动 fetch + memoize + 5s timeout 快速失败；
- `user:mcp_servers` scope 直接检查绕过 isClaudeAISubscriber bug；
- `vscodeSdkMcp` 反向通知：Claude Code 当 MCP server 给 VSCode 发文件更新；
- 与 swarm InProcessBackend 对比——同名字不同目的。

下一篇 → [05 client 核心：连接与 capability](./05-client核心-连接与capability.md)
