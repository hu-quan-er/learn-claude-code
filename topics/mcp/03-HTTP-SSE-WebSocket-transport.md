# 03 HTTP / SSE / WebSocket transport

> 3 种网络 transport，几百行的连接配置细节——本篇拆 SSE 双 fetch 设计（长连接 vs 短请求）、HTTP `Streamable HTTP` 协议要求、WebSocket node 实现差异、fresh-timeout-per-request 防 stale AbortSignal、Accept header 协议合规。

## 3.1 3 种网络 transport 概览

| transport | 模型 | 典型场景 |
|-----------|------|---------|
| **`sse`** | 单 HTTP GET 长连接 + 短 POST 命令 | 经典 MCP 网络部署 |
| **`http`** | Streamable HTTP（双向 POST，含 SSE 升级） | 新版 MCP 标准 |
| **`ws`** | WebSocket 双向 | 实时双向（少用） |

3 种各有 `*-ide` 子类（`sse-ide` / `ws-ide`）——为 IDE 扩展专用，没有 OAuth 流程。

代码集成：

```ts
import {
  SSEClientTransport,
  type SSEClientTransportOptions,
} from '@modelcontextprotocol/sdk/client/sse.js'
import {
  StreamableHTTPClientTransport,
  type StreamableHTTPClientTransportOptions,
} from '@modelcontextprotocol/sdk/client/streamableHttp.js'
```

`@modelcontextprotocol/sdk` 提供 transport class，Claude Code 配置它们。

## 3.2 全局 timeout 与超时常量

`client.ts:456-471`：

```ts
function getConnectionTimeoutMs(): number {
  return parseInt(process.env.MCP_TIMEOUT || '', 10) || 30000
}

const MCP_REQUEST_TIMEOUT_MS = 60000

const MCP_STREAMABLE_HTTP_ACCEPT = 'application/json, text/event-stream'
```

3 个数字：

| 常量 | 默认值 | 用途 |
|------|-------|------|
| `getConnectionTimeoutMs()` | 30s | server 整体**连接**超时（initial handshake） |
| `MCP_REQUEST_TIMEOUT_MS` | 60s | 每个 **请求**超时（auth / tool call / etc） |
| `MCP_STREAMABLE_HTTP_ACCEPT` | 字符串 | MCP Streamable HTTP 协议规定的 Accept header |

`MCP_TIMEOUT` 环境变量允许覆盖 30s 连接超时——某些 server 启动慢（OAuth flow 在浏览器里要时间）。

## 3.3 wrapFetchWithTimeout —— fresh-timeout-per-request

`client.ts:492+` 是个关键 helper：

```ts
export function wrapFetchWithTimeout(baseFetch: FetchLike): FetchLike {
  return async (url: string | URL, init?: RequestInit) => {
    const method = (init?.method ?? 'GET').toUpperCase()

    // Skip timeout for GET requests - long-lived SSE streams
    if (method === 'GET') {
      return baseFetch(url, init)
    }

    // POST / PUT / etc — 60s timeout per request
    const timeoutSignal = AbortSignal.timeout(MCP_REQUEST_TIMEOUT_MS)
    const composedSignal = init?.signal
      ? AbortSignal.any([init.signal, timeoutSignal])
      : timeoutSignal

    // Force Accept header for MCP Streamable HTTP spec
    const headers = new Headers(init?.headers)
    if (!headers.has('Accept')) {
      headers.set('Accept', MCP_STREAMABLE_HTTP_ACCEPT)
    }

    return baseFetch(url, { ...init, signal: composedSignal, headers })
  }
}
```

注释（`client.ts:473-484`）解释为什么需要：

> Wraps a fetch function to apply a **fresh timeout signal to each request**. This avoids the bug where a single `AbortSignal.timeout()` created at connection time becomes **stale after 60 seconds**, causing all subsequent requests to fail immediately with "The operation timed out."

经典 Web API 陷阱：

- `AbortSignal.timeout(60000)` 创建后**60 秒后过期**——之后任何使用它的请求都立刻 abort；
- 如果 transport 在连接时创建一次 signal、所有请求都用同一个——**60 秒后所有请求都立刻失败**；
- 修复：**每次请求**生成新 signal。

### Accept header 协议合规

注释另一段：

> MCP Streamable HTTP spec requires clients to advertise acceptance of both JSON and SSE on every POST. Servers that enforce this strictly reject requests without it (HTTP 406).
>
> The MCP SDK sets this inside StreamableHTTPClientTransport.send(), but it is attached to a Headers instance that passes through an object spread here, and some runtimes/agents have been observed dropping it before it reaches the wire. See https://github.com/anthropics/claude-agent-sdk-typescript/issues/202.
>
> Normalizing here (the last wrapper before fetch()) guarantees it is sent.

**协议规定**：POST 请求必须 `Accept: application/json, text/event-stream`。SDK 设了这个 header 但**某些 runtime spread 时会丢**——所以 Claude Code 在 fetch wrapper 里**重新强制设置**。

这是个**多层冗余**——上游设了下游可能丢，下游强制再设。

### GET 请求豁免

```ts
if (method === 'GET') {
  return baseFetch(url, init)
}
```

GET 在 MCP 是**长连接 SSE 流**——不能超时。注释明确：

> GET requests are excluded from the timeout since, for MCP transports, they are long-lived SSE streams meant to stay open indefinitely.

## 3.4 SSE transport —— 双 fetch 设计

`client.ts:619-677` 的 SSE transport 设置：

```ts
if (serverRef.type === 'sse') {
  const authProvider = new ClaudeAuthProvider(name, serverRef)
  const combinedHeaders = await getMcpServerHeaders(name, serverRef)

  const transportOptions: SSEClientTransportOptions = {
    authProvider,
    // POST 用的 fetch (带 timeout)
    fetch: wrapFetchWithTimeout(
      wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
    ),
    requestInit: {
      headers: {
        'User-Agent': getMCPUserAgent(),
        ...combinedHeaders,
      },
    },
  }

  // SSE 长连接用的 fetch (不带 timeout)
  transportOptions.eventSourceInit = {
    fetch: async (url, init) => {
      const authHeaders: Record<string, string> = {}
      const tokens = await authProvider.tokens()
      if (tokens) authHeaders.Authorization = `Bearer ${tokens.access_token}`

      const proxyOptions = getProxyFetchOptions()
      return fetch(url, {
        ...init,
        ...proxyOptions,
        headers: {
          'User-Agent': getMCPUserAgent(),
          ...authHeaders,
          ...init?.headers,
          ...combinedHeaders,
          Accept: 'text/event-stream',
        },
      })
    },
  }

  transport = new SSEClientTransport(new URL(serverRef.url), transportOptions)
}
```

注释（`client.ts:643-647`）的核心设计：

> IMPORTANT: Always set eventSourceInit with a fetch that does NOT use the timeout wrapper. The EventSource connection is long-lived (stays open indefinitely to receive server-sent events), so applying a 60-second timeout would kill it. The timeout is only meant for individual API requests (POST, auth refresh), not the persistent SSE stream.

**两个不同的 fetch**：

| fetch | 用途 | timeout |
|-------|------|---------|
| `transportOptions.fetch` | POST 短请求（命令） | 60s 强制 |
| `transportOptions.eventSourceInit.fetch` | GET 长连接（接收 server events） | **无** |

如果用同一个 fetch wrapper——60s 后 SSE 长连接被强杀 → server events 收不到 → 用户体验完全坏掉。

### 双 fetch 各自的 Accept

注意：

```ts
// POST fetch (wrapFetchWithTimeout 内部):
'Accept': 'application/json, text/event-stream'  // MCP 协议要求

// GET fetch (eventSourceInit):
'Accept': 'text/event-stream'  // SSE 协议要求
```

两种 Accept 不一样——但都符合各自协议。

### `wrapFetchWithStepUpDetection`

```ts
fetch: wrapFetchWithTimeout(
  wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
),
```

中间层 `wrapFetchWithStepUpDetection`——检测 server 返回的 OAuth step-up 挑战（"需要更高权限的 token"）。详见 [07 OAuth 认证](./07-OAuth与3套认证体系.md)。

包装顺序很关键：**step-up 在内层**——必须在 SDK 看到 403 之前先识别，否则 SDK 会用旧 token 重试导致死循环。

## 3.5 HTTP transport —— Streamable HTTP

`client.ts:798-865` 的 HTTP transport：

```ts
const authProvider = new ClaudeAuthProvider(name, serverRef)
const combinedHeaders = await getMcpServerHeaders(name, serverRef)

// 看是否已有 OAuth tokens
const hasOAuthTokens = !!(await authProvider.tokens())

const proxyOptions = getProxyFetchOptions()

const transportOptions: StreamableHTTPClientTransportOptions = {
  authProvider,
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
  ),
  requestInit: {
    ...proxyOptions,
    headers: {
      'User-Agent': getMCPUserAgent(),
      // 双层 Authorization 互斥逻辑
      ...(sessionIngressToken &&
        !hasOAuthTokens && {
          Authorization: `Bearer ${sessionIngressToken}`,
        }),
      ...combinedHeaders,
    },
  },
}

transport = new StreamableHTTPClientTransport(
  new URL(serverRef.url),
  transportOptions,
)
```

几个关键点：

### 1. OAuth 和 ingress token 互斥

```ts
...(sessionIngressToken &&
  !hasOAuthTokens && {
    Authorization: `Bearer ${sessionIngressToken}`,
  }),
```

**只在没有 OAuth tokens 时**才设置 ingress token。注释（`client.ts:807-811`）：

> Check if this server has stored OAuth tokens. If so, the SDK's authProvider will set Authorization — don't override with the session ingress token (SDK merges requestInit AFTER authProvider).
>
> CCR proxy URLs (ccr_shttp_mcp) have no stored OAuth, so they still get the ingress token. See PR #24454 discussion.

SDK 的 merge 顺序：authProvider 设的 header 先 → requestInit 后 merge 覆盖。如果 requestInit 也设 Authorization，会覆盖 authProvider 的——OAuth token 失效。

**互斥**：有 OAuth 就用 OAuth、没 OAuth 才用 ingress token。引用 PR #24454——这是个修过的真实 bug。

### 2. headers 日志脱敏

```ts
const headersForLogging = transportOptions.requestInit?.headers
  ? mapValues(
      transportOptions.requestInit.headers as Record<string, string>,
      (value, key) =>
        key.toLowerCase() === 'authorization' ? '[REDACTED]' : value,
    )
  : undefined
```

打 debug log 时**Authorization header 替换成 [REDACTED]**——避免 token 出现在日志文件里。

`toLowerCase()` 是因为 HTTP headers 大小写不敏感——某些库返回小写、某些 PascalCase。

### 3. Streamable HTTP 是什么

[Streamable HTTP spec](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports) 是 MCP 2025-03-26 版引入的新 transport：

- 同一个 HTTP endpoint 既能 POST 命令也能升级成 SSE 流；
- 比 SSE（双 endpoint）简化；
- 支持 session header 让 server 状态可恢复。

Claude Code 把它当 `http` type 实现——一个 endpoint 搞定所有。

## 3.6 WebSocket transport

`client.ts:708+` 的 WebSocket 处理比 SSE/HTTP 复杂——因为 Node.js 没有内置 WebSocket client：

```ts
} else if (serverRef.type === 'ws-ide') {
  const tlsOptions = getWebSocketTLSOptions()
  const wsHeaders = {
    'User-Agent': getMCPUserAgent(),
    ...(serverRef.authToken && {
      'X-Claude-Code-Ide-Authorization': serverRef.authToken,
    }),
  }

  let wsClient: WsClientLike
  if (typeof Bun !== 'undefined') {
    // Bun 内置 WebSocket
    wsClient = new WebSocket(serverRef.url, { /* ... */ })
  } else {
    // Node.js: 用 ws 库
    wsClient = await createNodeWsClient(serverRef.url, {
      headers: wsHeaders,
      ...tlsOptions,
    })
  }

  // 自定义 transport 包装 ws client
  transport = createMcpWebSocketTransport(wsClient)
}
```

### Bun vs Node 分支

```ts
if (typeof Bun !== 'undefined') {
  wsClient = new WebSocket(serverRef.url, { /* ... */ })
} else {
  wsClient = await createNodeWsClient(serverRef.url, { ... })
}
```

**Bun 运行时**有内置 WebSocket。**Node.js** 需要从 `ws` npm package 加载。Claude Code 支持两个 runtime（Bun 是 ant 内部主力），所以双分支。

### WsClientLike interface

```ts
type WsClientLike = {
  readonly readyState: number
  close(): void
  send(data: string): void
}
```

最小接口——只要支持 readyState/close/send 就行。让 Bun WebSocket 和 ws package 的 `WebSocket` 都能符合。

### createNodeWsClient

`client.ts:436-447`：

```ts
async function createNodeWsClient(
  url: string,
  options: Record<string, unknown>,
): Promise<WsClientLike> {
  const wsModule = await import('ws')
  const WS = wsModule.default as unknown as new (
    url: string,
    protocols: string[],
    options: Record<string, unknown>,
  ) => WsClientLike
  return new WS(url, ['mcp'], options)
}
```

3 个细节：

1. **动态 import**：只在需要时加载 `ws` package——`stdio` server 不需要 WebSocket 依赖；
2. **`['mcp']` protocol**：WebSocket sub-protocol 字段，告诉 server "我说 MCP 协议"；
3. **cast 构造函数**：注释（`client.ts:432-434`）说"Bun's ws shim types lack the 3-arg constructor that the real ws package supports, so we cast"——Bun 的 type 不全，所以强转。

### TLS options

`getWebSocketTLSOptions()` 提供 SSL/TLS 配置——支持自签证书等内部环境。

## 3.7 WebSocket vs SSE 的根本区别

| 维度 | SSE / Streamable HTTP | WebSocket |
|------|----------------------|-----------|
| 底层协议 | HTTP/1.1 | HTTP/1.1 升级到 WS |
| 数据方向 | server → client 单向（POST 命令是独立请求） | 全双工 |
| 帧格式 | text/event-stream（`data:` 行） | 二进制或文本帧 |
| 浏览器支持 | 原生 `EventSource` | 原生 `WebSocket` |
| Node.js | 不需要额外库 | 需要 `ws` package |
| 中间代理 | HTTP proxy 通常 OK | 某些 proxy 不支持 WS upgrade |

**SSE 在不支持 WebSocket 的环境（某些企业 proxy）能用**——这是它仍然是主流的原因。

但 WebSocket **双向**——server 主动 push（不只是 events）。MCP 的 elicitation（server 反向问 user）天然适合 WS。

## 3.8 mcpWebSocketTransport.ts —— Claude Code 自己实现 WS transport

`utils/mcpWebSocketTransport.ts` 200 行——Claude Code 自己实现 `Transport` 接口包装 WebSocket：

```ts
// 推断结构
export function createMcpWebSocketTransport(ws: WsClientLike): Transport {
  let onMessage: ((message: JSONRPCMessage) => void) | undefined
  let onError: ((error: Error) => void) | undefined
  let onClose: (() => void) | undefined

  ws.on?.('message', (data) => {
    try {
      const message = jsonParse(data.toString())
      onMessage?.(message)
    } catch (err) {
      onError?.(err)
    }
  })

  ws.on?.('error', (err) => onError?.(err))
  ws.on?.('close', () => onClose?.())

  return {
    send: async (message) => {
      ws.send(jsonStringify(message))
    },
    close: async () => {
      ws.close()
    },
    set onmessage(fn) { onMessage = fn },
    set onerror(fn) { onError = fn },
    set onclose(fn) { onClose = fn },
  }
}
```

为什么 SSE/HTTP 用 SDK 提供的 transport 但 WS 要自己实现？因为 SDK 的 ws transport 假设浏览器环境——Node.js 需要适配。

## 3.9 sse-ide 和 ws-ide —— IDE 扩展专用

回顾 [01](./01-McpServerConfig与8种transport-schema.md) 提过 `sse-ide` / `ws-ide` schema。它们和普通 sse/ws 的区别：

```ts
} else if (serverRef.type === 'sse-ide') {
  // IDE servers don't need authentication
  const transportOptions: SSEClientTransportOptions =
    proxyOptions.dispatcher
      ? { eventSourceInit: { ... } }
      : {}

  transport = new SSEClientTransport(
    new URL(serverRef.url),
    Object.keys(transportOptions).length > 0
      ? transportOptions
      : undefined,
  )
}
```

注释明确："IDE servers don't need authentication"——IDE 扩展和 Claude Code 在同一台机器，通过 lockfile 等机制建立信任，**不需要 OAuth**。

`ws-ide` 类似——但用 `authToken`（IDE 启动时生成的本地 token）：

```ts
const wsHeaders = {
  ...(serverRef.authToken && {
    'X-Claude-Code-Ide-Authorization': serverRef.authToken,
  }),
}
```

简化的认证模型——本地共享 secret。

## 3.10 proxy 支持

`getProxyFetchOptions()` 返回 proxy 配置（如果用户设了 `HTTPS_PROXY` 等）：

```ts
const proxyOptions = getProxyFetchOptions()
// → { dispatcher: ProxyAgent | undefined }
```

3 种 transport 都通过 `dispatcher` 选项支持 proxy——网络流量经过企业 proxy 后再到 MCP server。

**proxy 对 SSE 长连接的影响**：某些 proxy 有连接超时（如 60s 闲置后断开）。SSE 长连接需要 server 定期发 keepalive 才能存活。

## 3.11 几个隐性设计判断

### 1. fresh-timeout-per-request

避免 stale AbortSignal——每个 POST 新建 60s timeout。这是个**真实 bug 修复**——`AbortSignal.timeout()` 60 秒后过期是 Web Platform API 的已知行为。

### 2. SSE 双 fetch 分离 timeout 与非 timeout

EventSource 长连接不能超时——独立的 `eventSourceInit.fetch`。Claude Code 这个分离是必需的（不能用一个 wrapped fetch 走两种场景）。

### 3. Accept header 强制覆盖

MCP Streamable HTTP 协议要求 Accept header——SDK 设过但下游可能丢，**最后一层 fetch wrapper 强制设置**。多层冗余。

### 4. OAuth tokens 互斥 ingress token

PR #24454 修复——避免 SDK merge 顺序导致 OAuth 被 ingress token 覆盖。**显式互斥**而非依赖 merge 顺序。

### 5. Authorization 日志脱敏

debug log 里 `[REDACTED]`——保护 token。`toLowerCase()` 处理 header 大小写差异。

### 6. WebSocket 双 runtime 支持

Bun 内置 WebSocket vs Node 需要 `ws` package——动态分支。**支持两个 runtime** 是 ant 内部 Bun + 外部 Node 共存的实际需求。

### 7. WsClientLike 最小接口

只要 readyState/close/send——让任何 WebSocket 实现都能 fit。**最薄抽象**避免依赖具体实现。

### 8. 动态 import ws package

按需加载——非 WS 用户不需要这个依赖。**dependency 优化**。

### 9. WebSocket sub-protocol `['mcp']`

告诉 server "我说 MCP 协议"——让 server 拒绝其它客户端。

### 10. IDE transport 简化认证

`sse-ide` / `ws-ide` 不走 OAuth——本地共享 secret 足够。**信任模型按场景调整**——不是所有 transport 都需要同样严的认证。

### 11. 连接 vs 请求分离 timeout

`getConnectionTimeoutMs()` 30s（连接阶段） vs `MCP_REQUEST_TIMEOUT_MS` 60s（每个请求）。不同阶段不同 timeout——精细控制。

## 3.12 与 [bashtool] HTTP 路径对比

| 维度 | bashtool 跑 curl | MCP HTTP transport |
|------|-----------------|---------------------|
| 抽象层 | 子进程 + shell 命令 | SDK class + JSON-RPC |
| 认证 | curl `--header Authorization: ...` | OAuth provider |
| 长连接 | 不擅长 | SSE / WS 原生支持 |
| 重试 | 用户自己写 shell 循环 | transport 自带 reconnect |
| 跨平台 | 依赖 curl | Node.js fetch |

MCP HTTP 是**结构化的网络 IPC**——比 bashtool 跑 curl 高一个抽象层。

## 3.13 小结

- 3 种网络 transport：SSE / Streamable HTTP / WebSocket，加 sse-ide / ws-ide 子类；
- `wrapFetchWithTimeout` 每请求新 timeout 避免 stale AbortSignal；
- Accept header 强制设置——多层冗余保协议合规；
- SSE 双 fetch 设计：POST 用 timeout、GET 长连接不 timeout；
- HTTP transport 用 Streamable HTTP——同 endpoint POST/SSE 双模；
- OAuth tokens 和 ingress token **显式互斥**——避免 SDK merge 顺序 bug；
- Authorization debug log 脱敏 `[REDACTED]`；
- WebSocket Bun vs Node 双 runtime 分支；
- 动态 import `ws` 按需加载；
- WsClientLike 最小接口让多种 WebSocket 实现 fit；
- IDE transport 简化认证——本地共享 secret 不需要 OAuth；
- 连接 timeout (30s) vs 请求 timeout (60s) 分离。

下一篇 → [04 SDK 与 InProcess transport](./04-SDK与InProcess-transport.md)
