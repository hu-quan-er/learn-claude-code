# MCP 协议层集成专题

> 基于 `claude-code@2.1.88` 还原源码（`claude-code-sourcemap/restored-src/`）。
> 行号对应当前快照；版本升级会漂移。

## 阅读顺序

| # | 文档 | 主题 | 你会学到 |
|---|---|---|---|
| 00 | [总览与代码地图](./00-总览与代码地图.md) | 总览 | 16K 行分区、MCP 协议简介、5 种 capability、与已有专题关系 |
| 01 | [McpServerConfig 与 8 种 transport schema](./01-McpServerConfig与8种transport-schema.md) | 配置 | `types.ts` 8 个 schema、config.ts 1578 行：scope 合并、env 展开、7 级配置优先级 |
| 02 | [stdio 与子进程 transport](./02-stdio与子进程transport.md) | transport ① | stdio 启动子进程、JSON-RPC 帧解析、环境变量传递、崩溃恢复 |
| 03 | [HTTP / SSE / WebSocket transport](./03-HTTP-SSE-WebSocket-transport.md) | transport ② | 3 种网络 transport、connect / reconnect 策略、超时与重试 |
| 04 | [SDK 与 InProcess transport](./04-SDK与InProcess-transport.md) | transport ③ | `SdkControlTransport` / `InProcessTransport`、SDK MCP no-prefix、Claude.ai proxy |
| 05 | [client 核心：连接与 capability](./05-client核心-连接与capability.md) | 客户端 | `client.ts` 3348 行：`connectToServer` / `fetchTools/Resources/Commands`、能力协商、Unicode 清洗 |
| 06 | [MCPTool 代理与命名前缀](./06-MCPTool代理与命名前缀.md) | 工具集成 | `tools/MCPTool` 1086 行：tool result 转发、`mcp__server__tool` 命名、与权限对齐、`isResultTruncated` |
| 07 | [OAuth 与 3 套认证体系](./07-OAuth与3套认证体系.md) | 认证 | `auth.ts` 2465 行：标准 OAuth PKCE、Claude.ai proxy、xaa IdP、token 存储与刷新 |
| 08 | [MCPB bundle 与 plugin 集成](./08-MCPB-bundle与plugin集成.md) | 分发 | `mcpbHandler.ts` 968 行：`.mcpb` 文件格式、签名验证、`mcpPluginIntegration` 634 行 |
| 09 | [channel 抽象与 elicitation](./09-channel抽象与elicitation.md) | 元事件 | `channelNotification/Permissions/Allowlist` 632 行、`elicitationHandler` 313 行：MCP server → user 反向交互 |
| 10 | [连接管理与生命周期](./10-连接管理与生命周期.md) | 运行时 | `useManageMCPConnections` 1141 行、`prefetchAllMcpResources`、reconnect 策略、错误兜底 |
| 11 | [端到端模拟与设计哲学](./11-端到端模拟与设计哲学.md) | 收尾 | 一次完整"配置 → 连接 → 调用"推演 + 与 [swarm](../swarm/) / [bashtool](../bashtool/) 的 IPC 对比 + 设计哲学 |

## 一句话总结这个专题

MCP（Model Context Protocol）是 Anthropic 提出的**让 LLM 工具能被外部 server 提供**的协议——服务器声明工具、资源、prompt、elicitation 能力；客户端连上 server 把这些注入到 LLM 的上下文。

Claude Code 的 MCP 集成 **~16000 行代码**——8 种 transport / 3 套认证 / 5 种 capability / 多 server 并发管理 / OAuth + xaa + Claude.ai proxy / .mcpb bundle 分发——是这个项目里规模最大、最异构的子系统之一。

读完这个专题，你会理解：

- **MCP 是协议，不是 SDK**——同一 client 能连任何符合协议的 server（不限于 Anthropic 自己的）；
- 为什么 8 种 transport 都共存——每种解决不同部署场景（本地 stdio、远程 HTTP、IDE 内嵌 WebSocket、SDK 进程内、Claude.ai 代理...）；
- `mcp__server__tool` 命名前缀是怎么和 Claude Code 权限系统对齐的；
- 3 套认证（OAuth / Claude.ai / xaa）为什么必须分开——服务的群体不同；
- `prefetchAllMcpResources` 怎么解决启动时 N+1 个 server 串行连接的延迟问题；
- `.mcpb` bundle 怎么把 server 二进制和 manifest 打包成一个可分发文件；
- elicitation 让 MCP server **反向**询问用户——把 LLM agent 从"调工具的人"变成"被工具问问题的人"。

## 关键源码地图（16K 行）

| 模块 | 文件 | 行数 |
|------|------|----:|
| **核心客户端** | `services/mcp/client.ts` | **3348** |
| **认证** | `services/mcp/auth.ts` | **2465** |
| **配置** | `services/mcp/config.ts` | **1578** |
| **MCPB 加载** | `utils/plugins/mcpbHandler.ts` | 968 |
| **React 连接管理** | `services/mcp/useManageMCPConnections.ts` | 1141 |
| **plugin 集成** | `utils/plugins/mcpPluginIntegration.ts` | 634 |
| **MCPTool collapse** | `tools/MCPTool/classifyForCollapse.ts` | 604 |
| **utils** | `services/mcp/utils.ts` | 575 |
| **xaa IdP** | `services/mcp/xaa.ts` + `xaaIdpLogin.ts` | 998 |
| **MCPTool UI** | `tools/MCPTool/UI.tsx` | 402 |
| **slash command** | `commands/mcp/mcp.tsx` + `cli/handlers/mcp.tsx` | 445 |
| **channel 抽象** | `channelNotification.ts` + `channelPermissions.ts` + `channelAllowlist.ts` | 632 |
| **elicitation** | `elicitationHandler.ts` | 313 |
| **types** | `services/mcp/types.ts` | 258 |
| **mcp entrypoint** | `entrypoints/mcp.ts` | 196 |
| **WebSocket transport** | `utils/mcpWebSocketTransport.ts` | 200 |
| **output storage** | `mcpOutputStorage.ts` + `mcpValidation.ts` + `mcpInstructionsDelta.ts` | 527 |
| **SDK / InProcess / claudeai** | `SdkControlTransport.ts` + `InProcessTransport.ts` + `claudeai.ts` | 363 |
| **vscode SDK** | `vscodeSdkMcp.ts` | 112 |
| **MCPTool 主入口** | `tools/MCPTool/MCPTool.ts` | 77 |
| **utils 周边** | `headersHelper` + `mcpStringUtils` + `oauthPort` + `officialRegistry` + 其它 | ~600 |
| **合计** | | **~16000** |

## 与其它专题的关系

- **[bashtool 专题](../bashtool/)**：BashTool 也跑外部进程——但 stdio MCP 用结构化协议而不是 shell 命令；
- **[swarm 专题](../swarm/)**：两者都做 IPC——swarm 用文件系统，MCP 用各种 transport（stdio/SSE/WebSocket 等）；
- **[messages-pipeline 07](../messages-pipeline/07-attachment-normalizer.md)**：messages-pipeline 提到 `mcp_resource` / `mcp_instructions_delta` 等 attachment 类型——本专题展开它们的来源；
- **[core-models 02 Tool](../core-models/02-Tool-工具的统一抽象.md)**：MCPTool 是 Tool 统一抽象的具体实例；
- **[prompt-injection 03 Unicode 清洗](../prompt-injection/03-Unicode清洗管线.md)**：MCP server 返回的 tool list 走 Unicode 清洗——防止恶意 server 通过工具描述注入；
- **[overview 03 服务层](../../overview/03-服务层与API通信.md)**：MCP 是 services/ 下最大的子系统。

下一篇 → [00 总览与代码地图](./00-总览与代码地图.md)
