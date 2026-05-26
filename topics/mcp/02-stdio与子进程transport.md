# 02 stdio 与子进程 transport

> 最简单也最常用的 transport——启动一个本地子进程，通过 stdin/stdout JSON-RPC 帧通信。本篇拆 `StdioClientTransport` 集成、stderr 处理、shell prefix 注入、3 阶段优雅 shutdown（SIGINT → SIGTERM → SIGKILL）、进程崩溃恢复。

## 2.1 stdio 模型

stdio transport 的核心模型：

```
        Claude Code (parent process)
                │
                │ stdin  ──→  ┌────────────────────┐
                │              │  MCP Server          │
                │              │  (child process)     │
                │ stdout ←──  │  via JSON-RPC frames │
                │              └────────────────────┘
                │ stderr ←──  (logged to debug)
```

**Claude Code 启动子进程**——通过 `command + args + env` 配置。子进程通过 stdout 发 JSON-RPC 消息，stdin 接收 client 命令。stderr 用作 debug 输出（不参与协议）。

这是 MCP 协议**最早设计**的 transport——简单、本地、不需要网络、不需要认证。绝大多数 open-source MCP server（GitHub / Filesystem / Brave Search 等）都用 stdio。

## 2.2 StdioClientTransport —— `@modelcontextprotocol/sdk` 提供

Claude Code 不自己实现 stdio 帧解析——直接用 MCP SDK 的 `StdioClientTransport`：

```ts
// client.ts:12
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'
```

SDK 内部处理：

- 启动 child process（`child_process.spawn`）；
- 读 stdout 流，按 newline 分帧；
- 解析每行为 JSON-RPC message；
- 写 stdin 时把 JSON 消息序列化加 newline；
- 暴露 stderr 流给 caller。

Claude Code 只需要**配置和生命周期**——具体协议解析由 SDK 完成。

## 2.3 stdio transport 创建（client.ts:944-958）

```ts
} else if (serverRef.type === 'stdio' || !serverRef.type) {
  const finalCommand =
    process.env.CLAUDE_CODE_SHELL_PREFIX || serverRef.command
  const finalArgs = process.env.CLAUDE_CODE_SHELL_PREFIX
    ? [[serverRef.command, ...serverRef.args].join(' ')]
    : serverRef.args
  transport = new StdioClientTransport({
    command: finalCommand,
    args: finalArgs,
    env: {
      ...subprocessEnv(),
      ...serverRef.env,
    } as Record<string, string>,
    stderr: 'pipe',
  })
}
```

7 个细节：

### 1. `!serverRef.type` 也走 stdio

```ts
if (serverRef.type === 'stdio' || !serverRef.type) {
```

**老配置可能没 `type` 字段**——默认当 stdio。schema 里 `type` 是 optional 也是这个原因（向后兼容）。

### 2. `CLAUDE_CODE_SHELL_PREFIX` 注入

```ts
const finalCommand =
  process.env.CLAUDE_CODE_SHELL_PREFIX || serverRef.command
const finalArgs = process.env.CLAUDE_CODE_SHELL_PREFIX
  ? [[serverRef.command, ...serverRef.args].join(' ')]
  : serverRef.args
```

如果设了 `CLAUDE_CODE_SHELL_PREFIX`（如 `/bin/bash -c`），原命令变成它的参数。例：

- 原 config: `command: "npx", args: ["-y", "@github/server"]`
- 设了 `CLAUDE_CODE_SHELL_PREFIX="/bin/bash -c"`
- 实际启动: `/bin/bash -c "npx -y @github/server"`

用途：**在 shell wrapper 里跑 MCP server**——某些环境需要先 `source ~/.nvm/nvm.sh` 才能拿到 `npx`。用户用 wrapper 解决。

### 3. `subprocessEnv()` 剥敏感凭据

```ts
env: {
  ...subprocessEnv(),    // ← 已剥 ANTHROPIC_API_KEY / AWS_* / OAuth tokens 等
  ...serverRef.env,       // 然后 merge 用户配的
}
```

[bashtool 08](../bashtool/08-执行层与环境.md) 讲过的 `subprocessEnv` —— 在 GHA 等敏感场景下剥离凭据后再传子进程。MCP server 也走同样的 env 隔离——**MCP server 是外部代码，不该看到 Claude Code 自己的 API key**。

用户在 `serverRef.env` 配的（如 `GITHUB_TOKEN=${GITHUB_TOKEN}`）会覆盖到 subprocessEnv 之上——用户明确指定的优先。

### 4. `stderr: 'pipe'`

```ts
stderr: 'pipe',  // prevents error output from the MCP server from printing to the UI
```

`'pipe'` mode 把 stderr 接到 Node.js stream——而不是直接打印到终端。**避免 MCP server 的 debug 输出污染 Claude Code 的 UI**。

stderr 内容由 Claude Code 收集到 `stderrOutput` 变量，仅在 debug 时显示。

## 2.4 stderr 收集（client.ts:963-983）

```ts
// Set up stderr logging for stdio transport before connecting
let stderrHandler: ((data: Buffer) => void) | undefined
let stderrOutput = ''
if (serverRef.type === 'stdio' || !serverRef.type) {
  const stdioTransport = transport as StdioClientTransport
  if (stdioTransport.stderr) {
    stderrHandler = (data: Buffer) => {
      // Cap stderr accumulation to prevent unbounded memory growth
      if (stderrOutput.length < 64 * 1024 * 1024) {
        try {
          stderrOutput += data.toString()
        } catch {
          // Ignore errors from exceeding max string length
        }
      }
    }
    stdioTransport.stderr.on('data', stderrHandler)
  }
}
```

3 个细节：

### 1. 在 connect 之前注册

注释：

> Set up stderr logging for stdio transport before connecting in case there are any stderr outputs emitted during the connection start

如果 server 启动时立即出错（如缺依赖），错误输出在 stderr——必须**先注册 listener** 才不会漏。

### 2. 64MB 上限

```ts
if (stderrOutput.length < 64 * 1024 * 1024) {
```

stderr 累积上限 64 MB——防止恶意/卡死的 server 一直往 stderr 写撑爆内存。

### 3. catch 字符串大小溢出

```ts
try {
  stderrOutput += data.toString()
} catch {
  // Ignore errors from exceeding max string length
}
```

V8 的字符串有最大长度（~512MB 在 64-bit 上）。64MB 上限以下理论不会触发，但 `data.toString()` 内部可能因为 buffer 编码问题报错——catch 兜底。

## 2.5 ListRoots —— Claude Code 回复 server 询问的工作目录

```ts
client.setRequestHandler(ListRootsRequestSchema, async () => {
  return {
    roots: [
      {
        uri: `file://${getOriginalCwd()}`,
      },
    ],
  }
})
```

MCP 协议有个**反向请求** `roots/list`——server 询问 client "你的工作目录在哪？"。Claude Code 回复 `file://<originalCwd>`。

为什么 server 要知道 cwd？常见用例：

- Filesystem server 用 cwd 当 root（限制访问范围）；
- Git server 用 cwd 找 git 仓库根；
- 工程性的 MCP server 想知道项目位置。

**返回 `originalCwd` 而不是 `getCwd()`**——`originalCwd` 是 Claude Code 启动时的 cwd，session 中不变。如果用 `getCwd()`，模型 `cd` 后 server 看到的 cwd 跟着变，可能破坏 server 的状态。

## 2.6 connect 超时（client.ts:1020-1090）

```ts
// Add a timeout to connection attempts to prevent tests from hanging indefinitely
logMCPDebug(name, `Starting connection with timeout of ${getConnectionTimeoutMs()}ms`)

const connectPromise = client.connect(transport)
const timeoutPromise = new Promise<never>((_, reject) => {
  const timeoutId = setTimeout(() => {
    const elapsed = Date.now() - connectStartTime
    if (inProcessServer) {
      inProcessServer.close().catch(() => {})
    }
    transport.close().catch(() => {})
    reject(new TelemetrySafeError(
      `MCP server "${name}" connection timed out after ${getConnectionTimeoutMs()}ms`,
      'MCP connection timeout',
    ))
  }, getConnectionTimeoutMs())

  connectPromise.then(() => clearTimeout(timeoutId), () => clearTimeout(timeoutId))
})

try {
  await Promise.race([connectPromise, timeoutPromise])
  // ...
}
```

**race 模式**：connect 和 timeout 谁先完成。超时时 close transport 释放资源。

`getConnectionTimeoutMs()` 默认通常 30s——给 server 充分启动时间（`npx -y package` 第一次跑要下载、安装包）。

## 2.7 错误归因（client.ts:1290-1310）

连接失败时分类错误：

```ts
const transportType = serverRef.type || 'stdio'

if (error.message.includes('ENOENT')) {
  // 命令找不到
  throw new TelemetrySafeError(
    `Process not found - check that "${serverRef.command}" is installed and in PATH`,
    `MCP ${transportType} process not found`,
  )
} else if (error.message.includes('ECONNREFUSED')) {
  // 进程拒绝连接
} else if (error.message.includes('Process exited')) {
  // stdio server 启动后立刻退出
  throw new TelemetrySafeError(
    `Process not found - stdio server process terminated`,
    `MCP ${transportType} process terminated`,
  )
} else if (error.message.includes('spawn')) {
  throw new TelemetrySafeError(
    `Failed to spawn process - check command and permissions`,
    `MCP ${transportType} spawn failed`,
  )
}
```

不同错误类型 → 不同的用户友好消息：

| 错误特征 | 含义 | 用户怎么修 |
|---------|------|-----------|
| `ENOENT` | 命令找不到 | 检查 PATH / 装命令 |
| `Process exited` | 启动后立刻挂 | 看 stderr 找原因 |
| `spawn` | spawn 失败 | 检查权限 |
| 其它 | unknown | 看 debug 日志 |

`TelemetrySafeError` 让 analytics 上报错误类别但**不包含原始 error message**（可能含路径 / 用户信息）。

## 2.8 优雅 shutdown：SIGINT → SIGTERM → SIGKILL 3 阶段

`client.ts:1426-1540+` 是 stdio cleanup 最复杂的部分。注释：

> NOTE: StdioClientTransport.close() only sends an abort signal, but many MCP servers (especially Docker containers) need explicit SIGINT/SIGTERM signals to trigger graceful shutdown

SDK 的 `close()` 只是 abort signal——但 Docker / Python / Node 进程**默认不响应 abort**。需要 OS 级 signal。

### 3 阶段升级 kill

```ts
const childPid = stdioTransport.pid
if (childPid) {
  // 阶段 1: SIGINT (Ctrl+C)
  try {
    process.kill(childPid, 'SIGINT')
  } catch (error) {
    return
  }

  await new Promise<void>(async resolve => {
    let resolved = false

    // 监控进程是否退出 (50ms 间隔)
    const checkInterval = setInterval(() => {
      try {
        process.kill(childPid, 0)  // 0 = 检查存在不杀
      } catch {
        // Process 不存在 = 退出了
        if (!resolved) {
          resolved = true
          clearInterval(checkInterval)
          clearTimeout(failsafeTimeout)
          resolve()
        }
      }
    }, 50)

    // 600ms 绝对超时
    const failsafeTimeout = setTimeout(() => {
      if (!resolved) {
        resolved = true
        clearInterval(checkInterval)
        resolve()
      }
    }, 600)

    // 阶段 2: SIGTERM (100ms 后如果还没退)
    await sleep(100)
    if (!resolved) {
      try {
        process.kill(childPid, 0)
        // 还活着 → SIGTERM
        process.kill(childPid, 'SIGTERM')
      } catch {
        // 已退出
        return
      }

      // 阶段 3: SIGKILL (400ms 后还没退)
      await sleep(400)
      if (!resolved) {
        try {
          process.kill(childPid, 0)
          // 还活着 → SIGKILL (无视一切, 直接杀)
          process.kill(childPid, 'SIGKILL')
        } catch {
          // 已退出
        }
      }
    }
  })
}
```

**3 阶段升级**：

```
   T0:    发 SIGINT (Ctrl+C 等价)
   T+100ms: 还没退 → SIGTERM
   T+500ms: 还没退 → SIGKILL (无视一切)
   T+600ms: 超时兜底
```

每阶段间隔精心选定：

- **100ms SIGINT 等待**：正常 server 收到 SIGINT 几毫秒就退；
- **400ms SIGTERM 等待**：SIGTERM 用户能 trap，可能跑 cleanup 代码；
- **100ms 兜底**：SIGKILL 是 OS 强杀，不会失败；
- **总 600ms**：让 CLI 退出感觉响应。

`process.kill(pid, 0)` 是个**经典 trick**——signal 0 不实际杀，只检查进程存在。

### 为什么 Docker 特别需要

注释提到 Docker——Docker 容器是个 init process，不响应 abort signal。必须收到 SIGINT/SIGTERM 才会触发 `docker stop` 等价行为，让容器内进程优雅停。

如果 Claude Code 只调 `transport.close()` → SDK 发 abort → Docker 不理 → 进程留下来 → 累积成 zombie 容器。3 阶段 kill 是为这个场景设计的。

### 50ms 轮询 + 600ms failsafe

```ts
const checkInterval = setInterval(() => { ... }, 50)
const failsafeTimeout = setTimeout(() => { ... }, 600)
```

每 50ms 检查进程是否退出——比固定等 100ms 响应更快（正常进程几 ms 就退）。
600ms failsafe 是兜底——如果 OS 出问题（process.kill(0) 一直异常），不要让 cleanup 永远挂。

## 2.9 cleanup 流程

完整 cleanup 顺序：

```
1. 摘除 stderr listener  ← 防内存泄漏
2. SDK transport.close()  ← 发 abort signal
3. (stdio only) 3 阶段升级 kill 子进程
4. logMCPDebug 记录
```

第 1 步关键：

```ts
if (stderrHandler && (serverRef.type === 'stdio' || !serverRef.type)) {
  const stdioTransport = transport as StdioClientTransport
  stdioTransport.stderr?.off('data', stderrHandler)
}
```

**必须 off listener**——否则 closure 持有 transport 引用阻止 GC。这种 EventEmitter listener 内存泄漏是 Node.js 长期跑的常见问题。

## 2.10 isClaudeInChromeMCPServer / isComputerUseMCPServer —— 名字白名单走 in-process

`client.ts:905-943` 有个特殊路径——某些 stdio server **强制走 in-process**：

```ts
} else if (
  (serverRef.type === 'stdio' || !serverRef.type) &&
  isClaudeInChromeMCPServer(name)
) {
  // Run the Chrome MCP server in-process to avoid spawning a ~325 MB subprocess
  const { createChromeContext } = await import('../../utils/claudeInChrome/mcpServer.js')
  const { createClaudeForChromeMcpServer } = await import('@ant/claude-for-chrome-mcp')
  const { createLinkedTransportPair } = await import('./InProcessTransport.js')
  const context = createChromeContext(serverRef.env)
  inProcessServer = createClaudeForChromeMcpServer(context)
  const [clientTransport, serverTransport] = createLinkedTransportPair()
  await inProcessServer.connect(serverTransport)
  transport = clientTransport
} else if (
  feature('CHICAGO_MCP') &&
  (serverRef.type === 'stdio' || !serverRef.type) &&
  isComputerUseMCPServer!(name)
) {
  // Computer Use MCP server in-process — same rationale
  // ...
}
```

注释明确："avoid spawning a ~325 MB subprocess"。这两个 server 在 Node 进程里都有完整实现——配置成 stdio 是为了**协议兼容**（让模型把它当普通 MCP 看），但实际**跳过 spawn 走 in-process**。

这是个**性能优化**——Chrome MCP server 启动一个子进程要 325MB 内存（含 V8 + 依赖），共享主进程内存能节省很多。

[04 SDK 与 InProcess transport](./04-SDK与InProcess-transport.md) 详述这个机制。

## 2.11 ListRoots vs 工具实际 cwd

[1.7 节](#1.7-listroots) 提到 ListRoots 返回 `originalCwd`。但 stdio server 启动时**实际 cwd** 是什么？

```ts
new StdioClientTransport({
  command: finalCommand,
  args: finalArgs,
  env: { ... },
  // 没显式设 cwd!
})
```

`StdioClientTransport` 默认用**当前进程的 cwd**——也就是 `process.cwd()` 在 spawn 时的值。

这意味着：

- 用户 `cd` 到 `/tmp` 启动 Claude Code → MCP server 在 `/tmp` 启动；
- 之后 Claude Code session 里 `cd /project` → 已启动的 MCP server **不变**（它的 cwd 还是 `/tmp`）；
- 但通过 `ListRoots` 它收到 `file:///tmp`（originalCwd）—— **一致**。

所以 `originalCwd` 选择是**故意的**——和 server 启动时的实际 cwd 对齐。

## 2.12 几个隐性设计判断

### 1. type optional 向后兼容

老配置没 `type`——默认当 stdio。schema 显式 `optional` + 代码 `|| !serverRef.type` 检查。**给老用户无痛升级路径**。

### 2. CLAUDE_CODE_SHELL_PREFIX 提供 wrapper hook

让用户用 wrapper 解决"PATH 不全"等问题。**简单的 env var hook 解决一类用户痛点**——比加各种 shell 集成代码简洁。

### 3. subprocessEnv 剥凭据

MCP server 是外部代码，不应该看到 Claude Code 自己的 API key——通过 `subprocessEnv` 自动剥。

### 4. stderr 必须 pipe + listener 必须 off

`stderr: 'pipe'` 防止污染 UI；listener 必须 off 防内存泄漏。这两个细节是 Node.js EventEmitter 模型的标准坑。

### 5. ListRoots 返回 originalCwd 而不是当前 cwd

避免 server cwd 状态混乱——和 server 启动时的实际 cwd 对齐。这是 **stateful server 的一致性考虑**。

### 6. connect 超时用 Promise.race + clearTimeout

避免泄漏 timeout——`connectPromise.then(clear, clear)`。

### 7. 3 阶段升级 kill

SIGINT (100ms) → SIGTERM (400ms) → SIGKILL—— 每阶段精心选定，总 600ms。**为 Docker 等不响应 abort 的进程设计**。

### 8. process.kill(pid, 0) 检查存在

经典 trick——signal 0 不杀只检查。每 50ms 轮询比固定等更响应。

### 9. 64MB stderr 上限 + try/catch

防恶意/卡死 server 撑爆内存。V8 字符串大小有上限——catch 兜底。

### 10. ENOENT / spawn / Process exited 分类错误

不同错误类型给不同用户友好消息——`TelemetrySafeError` 让 analytics 知道分类但不泄漏 PII。

### 11. in-process 优化绕过 spawn

Chrome / Computer Use server 用配置成 stdio 但实际 in-process——节省 325MB+ 内存。**配置不变、实现变**——对用户透明。

## 2.13 与 [bashtool 08 执行层](../bashtool/08-执行层与环境.md) 对比

| 维度 | bashtool stdio | MCP stdio |
|------|--------------|-----------|
| 启动什么 | shell (bash/zsh) | MCP server (任意命令) |
| 通信内容 | 纯文本命令 / 输出 | JSON-RPC 帧 |
| stdin 内容 | 命令字符串 + Enter | JSON-RPC messages |
| stderr 处理 | 合并进 stdout (file mode) | 独立 pipe + 64MB 上限 |
| env 处理 | subprocessEnv + GIT_EDITOR=true + ... | subprocessEnv + serverRef.env |
| 生命周期 | 短 (命令完就退) | 长 (session 期间一直跑) |
| 超时机制 | DEFAULT_TIMEOUT 30 min | getConnectionTimeoutMs() 30s 连接超时 |
| 杀进程 | tree-kill 整棵进程树 | 3 阶段 SIGINT/TERM/KILL |
| 主要目的 | 跑用户命令 | 提供工具能力 |

**bashtool 是"一次性命令"，MCP stdio 是"长期 server"**——生命周期截然不同。

## 2.14 小结

- stdio transport 启动子进程通过 stdin/stdout 通信，stderr 单独 pipe；
- 用 SDK 的 `StdioClientTransport`，Claude Code 处理配置和生命周期；
- `CLAUDE_CODE_SHELL_PREFIX` env var 允许 wrapper 注入（解决 PATH 不全）；
- `subprocessEnv()` 剥敏感凭据 — MCP server 是外部代码；
- stderr 必须 pipe + 必须 off listener（内存）+ 64MB 上限（防失控）；
- ListRoots 反向请求返回 `originalCwd` 而不是 `getCwd()`，避免 server 状态混乱；
- 连接超时 race + clearTimeout 避免泄漏；
- 错误分类（ENOENT / spawn / Process exited）→ TelemetrySafeError 分类上报；
- 3 阶段优雅 shutdown：SIGINT (100ms) → SIGTERM (400ms) → SIGKILL，总 600ms；
- `process.kill(pid, 0)` 检查存在不杀；
- Chrome / Computer Use server 配置成 stdio 但实际 in-process——绕过 325MB 子进程；
- 与 bashtool stdio 模型对比：协议化 + 长期 vs 文本化 + 一次性。

下一篇 → [03 HTTP / SSE / WebSocket transport](./03-HTTP-SSE-WebSocket-transport.md)
