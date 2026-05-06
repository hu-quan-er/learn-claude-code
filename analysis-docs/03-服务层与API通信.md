# 03 - 服务层与 API 通信

---

## 一、API 通信层架构

### 1.1 多提供商支持

**文件**: `src/services/api/client.ts`

Claude Code 支持 4 个后端 API 提供商：

| 提供商 | SDK | 启用条件 |
|--------|-----|----------|
| **Anthropic 第一方** | `Anthropic` | 默认 |
| **AWS Bedrock** | `AnthropicBedrock` | `CLAUDE_CODE_USE_BEDROCK` |
| **Azure Foundry** | `AnthropicFoundry` | `CLAUDE_CODE_USE_FOUNDRY` |
| **Google Vertex AI** | `AnthropicVertex` | `CLAUDE_CODE_USE_VERTEX` |

**优先级**：`BEDROCK > FOUNDRY > VERTEX > 默认 API`

### 1.2 请求构建与发送

**文件**: `src/services/api/claude.ts`

关键特性：
- **Beta 头部管理**：通过 `anthropic_beta` 标头启用试验特性（Prompt Caching、Thinking、Structured Outputs 等）
- **自定义头部注入**：支持 `ANTHROPIC_CUSTOM_HEADERS` 环境变量
- **请求 ID 追踪**：自动生成 `x-client-request-id` 用于超时关联
- **User-Agent 自定义**：包含 CLI 版本、系统信息、运行时标识

### 1.3 流式响应处理

- 使用 SDK 的 `Stream<BetaMessage>` 类型
- 支持增量内容块（text、tool_use、text_delta 等）
- 处理工具调用流式返回
- Token 计数从 API 响应头提取

---

## 二、错误处理与重试机制

### 2.1 错误分类

**文件**: `src/services/api/errors.ts`, `src/services/api/errorUtils.ts`

| 错误类型 | 说明 |
|----------|------|
| 网络错误 | Connection、Timeout |
| SSL/TLS 错误 | 证书验证失败、自签名证书 |
| API 错误 | 429 限流、529 容量、401 认证、400 验证 |
| 业务逻辑错误 | Prompt Too Long、超配额 |
| CloudFlare 拦截 | 检测并提取 HTML 标题 |

特殊处理：
- 代理和企业环境 SSL 证书错误的友好提示
- 从嵌套 JSON 结构提取错误信息（支持 Bedrock 和标准 API 格式）

### 2.2 智能重试策略

**文件**: `src/services/api/withRetry.ts`

#### 重试决策
- 最多重试 **10 次**（可配置）
- 429/529 容量错误持续重试
- 401 认证错误重新获取客户端
- ECONNRESET/EPIPE 禁用 keep-alive 后重试

#### 前景 vs 后景差异化
```
前景查询（用户阻塞）：支持 529 重试
后景查询（摘要、建议等）：立即失败
理由：容量级联期间，每次重试放大 3-10 倍的网关负载
```

#### 指数退避
- 基础延迟：500ms
- 每次重试增加指数退避
- 最大持久重试延迟：5 分钟
- 支持 AFK 模式下的无限持久重试

#### 529 限制与模型降级
- 最多 3 次连续 529 错误，然后触发模型降级
- 主模型 529 连续 3 次 → 降级到备用模型
- 降级触发 `FallbackTriggeredError`

---

## 三、认证与授权机制

### 3.1 OAuth 2.0 PKCE 流程

**文件**: `src/services/oauth/index.ts`

```typescript
OAuthService 类实现：
- generateCodeVerifier() / generateCodeChallenge()  // PKCE 安全参数
- waitForAuthorization()  // 本地 HTTP 监听器自动捕获授权码

两种授权方式：
1. 自动流：打开浏览器，监听 localhost 重定向
2. 手动流：用户粘贴授权码（非浏览器环境）
```

### 3.2 API 密钥管理优先级

```
1. Claude.ai OAuth 令牌（isClaudeAISubscriber）
2. API Key Helper（交互式提示或 keychain 存储）
3. ANTHROPIC_AUTH_TOKEN 环境变量
4. ANTHROPIC_API_KEY 环境变量
```

### 3.3 MCP OAuth 认证

**文件**: `src/services/mcp/auth.ts`

- 支持 XAA（Cross-App Access）与自定义 IdP
- 支持 OAuth 令牌刷新机制
- 支持元数据发现（OpenID Connect）
- 安全存储在系统钥匙链

失败原因分类：
- `metadata_discovery_failed`
- `no_client_info`
- `invalid_grant`
- `transient_retries_exhausted`

---

## 四、模型选择与切换逻辑

### 4.1 模型选择优先级

**文件**: `src/utils/model/model.ts`

```
1. 会话期间模型覆盖（/model 命令）  ← 最高优先级
2. 启动时模型覆盖（--model 标志）
3. ANTHROPIC_MODEL 环境变量
4. 用户保存的设置
5. 内置默认模型                      ← 最低优先级
```

### 4.2 模型类别

| 类别 | 获取函数 | 特点 |
|------|----------|------|
| **Haiku** | `getSmallFastModel()` | 快速推理，令牌优化 |
| **Sonnet** | `getDefaultSonnetModel()` | 平衡速度和能力 |
| **Opus** | `getDefaultOpusModel()` | 最强能力 |
| **Custom** | 用户指定 | Bedrock/Vertex 3P 模型 |

### 4.3 模型能力检测

```typescript
modelSupportsThinking()           // Extended Thinking
modelSupportsAdaptiveThinking()   // 自适应思考
modelSupportsStructuredOutputs()  // 结构化输出
modelSupportsEffort()             // 努力级别（思考预算）
has1mContext()                    // 1M token 上下文窗口
```

---

## 五、MCP（模型上下文协议）集成

### 5.1 MCP 客户端管理

**文件**: `src/services/mcp/client.ts`

核心功能：
- 支持 7+ 种传输方式：stdio、SSE、WebSocket、HTTP、SDK、ClaudeAI 代理
- 工具缓存（LRU）防止重复发现
- 资源链接处理和验证
- 内容截断检测（超过上下文限制）

### 5.2 MCP 服务器配置类型

**文件**: `src/services/mcp/types.ts`

```typescript
McpStdioServerConfig       // 命令行工具
McpSSEServerConfig         // 服务器发送事件
McpHTTPServerConfig        // HTTP 服务
McpWebSocketServerConfig   // WebSocket 实时
McpSdkServerConfig         // SDK 集成（in-process）
McpClaudeAIProxyServerConfig  // Claude.ai 代理
```

### 5.3 配置范围（Scope）

```
local       → 项目级（.claude/config.json）
user        → 用户级（~/.claude/config.json）
project     → 项目级覆盖
dynamic     → 运行时动态
enterprise  → 企业托管
claudeai    → Claude.ai 代理服务器
managed     → 托管设置（只读）
```

### 5.4 工具规范化和验证

- 工具名称标准化（MCP 服务器返回原始名称，CLI 规范化）
- 描述长度上限：2048 字符（防止 OpenAPI 服务器垃圾数据）
- 内容截断检测（超过 context window）
- 二进制内容持久化到磁盘

---

## 六、上下文管理与 System Prompt 构建

### 6.1 System Prompt 优先级

**文件**: `src/context.ts`

```
1. 覆盖 System Prompt（-s 标志，完全替换）
2. 协调器 System Prompt（CLAUDE_CODE_COORDINATOR_MODE）
3. 代理 System Prompt（主线程代理定义）
4. 自定义 System Prompt（--system-prompt）
5. 默认 System Prompt
+ 追加 System Prompt（--append-system-prompt）
```

### 6.2 System Prompt 组成部分

```
- Claude Code 核心指令（决策、工具使用、安全）
- 工具目录（所有可用工具的描述）
- MCP 服务器指令（来自 server.instructions）
- 内存文件内容（从 ~/.claude/projects/<path>/memory）
- Git 状态（当前分支、最近提交、变更）
- 环境信息（OS、Node 版本、工作目录）
- 内存初始化提示（Session Memory）
```

### 6.3 Git 状态快照

- 在对话开始时捕获（对话期间不更新）
- 包含：当前分支、默认分支、未提交更改、最近 5 个提交
- 超过 2K 字符时截断，建议用户直接运行 git 命令

---

## 七、会话压缩（Compact）机制

### 7.1 压缩策略概览

**文件**: `src/services/compact/`

| 策略 | 文件 | 说明 |
|------|------|------|
| **传统压缩** | `compact.ts` | 启动分叉代理汇总对话历史 |
| **微压缩** | `microCompact.ts` | 清除旧工具结果（文件内容、命令输出等） |
| **缓存微压缩** | `cachedMicrocompact.ts` | 使用提示缓存进行增量压缩（ANT-only） |
| **自动压缩** | `autoCompact.ts` | 基于令牌阈值自动触发 |

### 7.2 传统压缩流程

```
触发器：手动 /compact 命令或自动压缩阈值
  → 执行预压缩钩子
  → 运行分叉代理（共享提示缓存）
  → 后处理
  → 执行后压缩钩子
  → 输出 CompactBoundaryMessage（标记压缩边界）
```

### 7.3 微压缩（Micro-Compact）

可压缩的工具结果：

| 工具 | 压缩内容 |
|------|----------|
| FileReadTool | 清除旧文件内容 |
| ShellTools | 清除旧命令输出 |
| GlobTool | 清除搜索结果 |
| GrepTool | 清除搜索结果 |
| WebSearchTool | 清除网页内容 |
| WebFetchTool | 清除网页内容 |
| FileEditTool | 清除编辑历史 |
| FileWriteTool | 清除写入历史 |

**时间基础配置**：
- 根据消息年龄自动清除（可配置天数）
- `TOKEN_BASED_MC_CLEARED_MESSAGE` 替代被清除的内容

### 7.4 自动压缩触发条件

```
- 连续查询计数阈值
- 令牌数量阈值
- 时间阈值
- 配置：gateCompactFeature() 通过 GrowthBook
```

---

## 八、LSP（Language Server Protocol）集成

**文件**: `src/services/lsp/`

### 8.1 LSP 服务器管理

```
单例模式：
- initializeLspServerManager()  // 启动期间初始化
- getLspServerManager()          // 获取实例或 undefined
- 初始化状态：not-started → pending → success/failed
- 异步初始化：不阻塞应用启动
```

### 8.2 功能

- 诊断（Diagnostics）：实时代码检查
- 悬停提示（Hover）：类型信息
- Go-to-Definition：符号导航
- 被动反馈：来自编辑器的后台信息

---

## 九、插件系统

### 9.1 插件操作

**文件**: `src/services/plugins/pluginOperations.ts`

```
核心操作：
- installPlugin()     — 安装到 local/project/user 范围
- uninstallPlugin()   — 卸载并清理
- enablePlugin()      — 切换激活状态
- updatePlugin()      — 检查和应用更新

依赖解析：
- findReverseDependents() — 识别依赖该插件的其他插件
- 卸载时检查反向依赖
```

### 9.2 插件作用域

```
user     → 仅对当前用户
project  → 当前项目目录及子目录
local    → 当前工作目录
managed  → 由企业管理设置安装（只读）
```

### 9.3 CLI 命令

```
plugin install <id>
plugin uninstall <id>
plugin enable/disable <id>
plugin list
plugin search <term>
```

---

## 十、分析/遥测系统

### 10.1 事件日志架构

**文件**: `src/services/analytics/index.ts`

设计原则：
- **无依赖**：避免循环导入
- **队列化**：在 sink 附加之前缓冲事件
- **路由**：同一个事件可发往多个后端

### 10.2 元数据卫生

```typescript
// 强制类型：
AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
// 防止代码片段、文件路径泄露
// 显式验证标记
```

### 10.3 事件 Sink

```
支持多个后端：
- Datadog（通用指标）
- 1P 事件日志（BigQuery，PII 受控列）
- GrowthBook（特性开关和实验配置）
```

---

## 十一、会话记忆服务

### 11.1 Session Memory

**文件**: `src/services/SessionMemory/`

自动维护 Markdown 笔记：记录关键决策、代码更改摘要、用户指定的重要信息。

执行方式：后台分叉代理（`runForkedAgent`），周期执行，不中断主对话流。

### 11.2 Extract Memories

**文件**: `src/services/extractMemories/`

触发时机：查询循环完成（模型返回无工具调用的最终响应）

```
机制：
- runForkedAgent() — 完全的会话分叉
- 共享提示缓存
- 提取持久记忆到 ~/.claude/projects/<path>/memory/
```

---

## 十二、提示缓存

### 12.1 缓存策略

**文件**: `src/services/api/promptCacheBreakDetection.ts`

```
getPromptCachingEnabled()：
- 全局禁用（DISABLE_PROMPT_CACHING）
- 按模型禁用（Haiku/Sonnet/Opus）
- 默认启用

TTL 策略：
- ephemeral type（临时缓存）
- 1 小时 TTL（如果查询源符合 GrowthBook allowlist）
```

### 12.2 缓存破坏检测

- 消息指纹变更检测
- 系统 prompt 变更记录

---

## 十三、速率限制处理

**文件**: `src/services/claudeAiLimits.ts`

限制类型：
- RPM（Requests Per Minute）
- TPM（Tokens Per Minute）
- 配额状态（从响应头提取）

Claude.ai 特定限制：
- 基于订阅等级
- 支持超额配额购买
- 订阅升级检查

---

## 十四、快速模式（Fast Mode）

**文件**: `src/utils/fastMode.ts`

```
- isFastModeEnabled()          // 用户激活
- isFastModeSupportedByModel() // 模型支持检查
- isFastModeCooldown()         // 容量冷却期
- triggerFastModeCooldown()    // 容量拒绝后冷却
```

---

## 总结：服务层架构特点

1. **多提供商支持** — 单一代码库支持第一方、Bedrock、Vertex、Foundry
2. **智能重试** — 基于查询来源的差异化重试策略，防止级联放大
3. **完整 OAuth** — OAuth 2.0 PKCE + MCP OAuth + XAA 跨应用访问
4. **灵活上下文** — 动态 system prompt 构建，支持 Git、内存、环境信息
5. **多级压缩** — 时间基础和令牌基础的自动会话压缩
6. **MCP 深度集成** — 支持 7+ 种传输方式、OAuth 认证、工具规范化
7. **分析驱动** — 事件驱动架构，PII 标记，多后端路由
8. **后台优化** — 分叉代理实现内存提取和会话总结，不阻塞主线程
