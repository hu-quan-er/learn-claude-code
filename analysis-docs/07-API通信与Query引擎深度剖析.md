# 07 - API 通信与 Query 引擎深度剖析

---

## 一、API 请求的完整构建过程

### 1.1 多层 Beta 特性聚合

**文件**: `src/services/api/claude.ts`

```typescript
const betas = getMergedBetas(options.model, { isAgenticQuery })
```

**Beta Header 优先级**：
1. 全局 Beta 头（Fast Mode、AFK Mode、Prompt Caching 等）
2. 模型特定 Beta（取决于型号版本支持）
3. 查询源特定 Beta（repl_main_thread vs compact vs sdk）

**关键 Beta 特性**：
| Beta Header | 功能 |
|-------------|------|
| `FAST_MODE_BETA_HEADER` | 快速模式（缓存优化） |
| `AFK_MODE_BETA_HEADER` | 无人值守模式（持续重试） |
| `PROMPT_CACHING_SCOPE_BETA_HEADER` | 全局缓存作用域 |
| `CACHE_EDITING_BETA_HEADER` | 缓存编辑（Ant 专用） |
| `TASK_BUDGETS_BETA_HEADER` | 令牌预算（EAP） |

### 1.2 系统提示构建与缓存标记

```typescript
const system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, {
  skipGlobalCacheForSystemPrompt: needsToolBasedCacheMarker,
  querySource: options.querySource,
})
```

**缓存控制策略**：
- **TTL 选择**：5 分钟（默认）vs 1 小时（需 Latch 状态）
- **作用域**：`ephemeral` type，可选 `scope: 'global'`
- **条件**：Ant 或订阅用户 + GrowthBook 白名单 + 会话稳定性

### 1.3 工具 Schema 构建

```typescript
const toolSchemas = await Promise.all(
  filteredTools.map(tool =>
    toolToAPISchema(tool, {
      getToolPermissionContext,
      tools,
      agents: options.agents,
      deferLoading: willDefer(tool),
    })
  )
)
```

**工具处理流程**：
1. 动态工具加载检测：`isDeferredTool()` 检查
2. MCP 工具延迟加载：`defer_loading: true` 标记
3. ToolSearch 集成：已知工具名称过滤
4. 权限检查：`getToolPermissionContext()` 异步验证

### 1.4 消息 API 化后处理

```typescript
// 1. 消息规范化
let messagesForAPI = normalizeMessagesForAPI(messages, filteredTools)

// 2. 工具引用块剥离（非 ToolSearch 模式）
messagesForAPI = messagesForAPI.map(msg => {
  return stripToolReferenceBlocksFromUserMessage(msg)
  return stripCallerFieldFromAssistantMessage(msg)
})

// 3. 修复工具配对
messagesForAPI = ensureToolResultPairing(messagesForAPI)

// 4. 裁剪媒体超限（最多 100 个图片+文档）
messagesForAPI = stripExcessMediaItems(messagesForAPI, 100)
```

---

## 二、流式响应处理的事件驱动架构

### 2.1 核心流式循环

**文件**: `src/query.ts`

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  signal: toolUseContext.abortController.signal,
  options: { model, fastMode, ... }
})) {
  if (message.type === 'assistant') {
    assistantMessages.push(message)

    // 提取 tool_use 块
    const toolUseBlocks = message.message.content.filter(
      content => content.type === 'tool_use'
    )
    if (toolUseBlocks.length > 0) {
      needsFollowUp = true
      for (const toolBlock of toolUseBlocks) {
        streamingToolExecutor.addTool(toolBlock, message)
      }
    }
  }
}
```

### 2.2 事件类型体系

| 类型 | 说明 |
|------|------|
| `StreamEvent` | 流式事件（content_block_delta、input_json_delta 等） |
| `AssistantMessage` | 完整助手消息 |
| `SystemAPIErrorMessage` | API 错误（重试通知） |
| `RequestStartEvent` | 请求开始信号 |
| `TombstoneMessage` | 消息撤销标记 |

### 2.3 中断与降级恢复

```typescript
if (streamingFallbackOccured) {
  // 发送墓碑消息，清除孤立消息
  for (const msg of assistantMessages) {
    yield { type: 'tombstone', message: msg }
  }
  // 重置工具执行器
  streamingToolExecutor = new StreamingToolExecutor(...)
}
```

---

## 三、Query 循环的状态机实现

### 3.1 核心状态

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking?: AutoCompactTrackingState
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  pendingToolUseSummary?: Promise<...>
  stopHookActive?: boolean
  turnCount: number
  transition?: Continue
}
```

### 3.2 Tool_Use → Execute → Continue 循环

```
┌──────────────────────────────────────────────┐
│ 1. API 调用 (queryModel)                      │
│    构建消息、系统提示、工具 Schema             │
│    流式接收响应                                │
└──────────────┬───────────────────────────────┘
               ↓
┌──────────────────────────────────────────────┐
│ 2. Tool Use 检测                              │
│    扫描 content 中 tool_use 块                │
│    收集 tool_use_id 和输入                    │
│    设置 needsFollowUp=true                    │
└──────────────┬───────────────────────────────┘
               ↓
┌──────────────────────────────────────────────┐
│ 3. Tool 执行 (runTools)                       │
│    权限检查 (canUseTool)                      │
│    并发/串行执行                              │
│    收集 tool_result 块                        │
└──────────────┬───────────────────────────────┘
               ↓
┌──────────────────────────────────────────────┐
│ 4. Continue 决策                              │
│    stop_reason 分析 →                         │
│    • max_output_tokens → 截断恢复             │
│    • ReactiveCompact → 反应性压缩             │
│    • StopHook → 重试                          │
│    • tool_use → 继续循环                      │
│    • end_turn → 返回完成                      │
└──────────────────────────────────────────────┘
```

### 3.3 Max_Output_Tokens 恢复

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

if (parseMaxTokensContextOverflowError(error)) {
  const { inputTokens, contextLimit } = overflowData
  const availableContext = contextLimit - inputTokens - 1000  // safety buffer

  const adjustedMaxTokens = Math.max(
    FLOOR_OUTPUT_TOKENS,              // 3000
    availableContext,
    thinkingConfig.budgetTokens + 1
  )
  retryContext.maxTokensOverride = adjustedMaxTokens
}
```

---

## 四、重试机制的精确算法

### 4.1 指数退避

**文件**: `src/services/api/withRetry.ts`

```typescript
function getRetryDelay(attempt, retryAfterHeader?, maxDelayMs = 32000) {
  // 服务器指导延迟（优先）
  if (retryAfterHeader) {
    return parseInt(retryAfterHeader, 10) * 1000
  }

  // 指数退避 + 25% 随机抖动
  const baseDelay = Math.min(
    500 * Math.pow(2, attempt - 1),  // 500ms * 2^(n-1)
    maxDelayMs                        // 上限 32s
  )
  return baseDelay + Math.random() * 0.25 * baseDelay
}
```

**延迟序列**：

| 尝试 | 范围 |
|------|------|
| 1 | 500ms - 625ms |
| 2 | 1s - 1.25s |
| 3 | 2s - 2.5s |
| 4 | 4s - 5s |
| 5+ | 32s（上限） |

### 4.2 529 过载处理与模型降级

```
快速模式 529 处理：
  ├─ 短延迟（retryAfter < 阈值）→ 等待后重试，保持 Fast Mode
  └─ 长延迟 → 进入冷却期（30分钟），切换标准模型

连续 529 降级：
  连续 3 次 529 → 触发 Model Fallback
    ├─ Ant 用户：Opus → Sonnet
    └─ 外部用户：抛出 CannotRetryError
```

### 4.3 无人值守模式（AFK Mode）

```typescript
const PERSISTENT_MAX_BACKOFF_MS = 5 * 60 * 1000   // 5 分钟最大单次延迟
const PERSISTENT_RESET_CAP_MS = 6 * 60 * 60 * 1000 // 6 小时绝对上限
const HEARTBEAT_INTERVAL_MS = 30_000                // 30 秒心跳

if (isPersistentRetryEnabled()) {
  // 持续重试 429/529（不限次数）
  // 分块延迟（每 30s 发送心跳消息保活会话）
  while (remaining > 0) {
    yield createSystemAPIErrorMessage(error, remaining, attempt, maxRetries)
    const chunk = Math.min(remaining, HEARTBEAT_INTERVAL_MS)
    await sleep(chunk, signal)
    remaining -= chunk
  }
}
```

### 4.4 认证刷新

```
401 → handleOAuth401Error() → 刷新令牌 → 获取新客户端
403 "token revoked" → 重新认证
ECONNRESET/EPIPE → 禁用 keep-alive → 重试
```

---

## 五、提示缓存的工作原理

### 5.1 缓存一致性检测

**文件**: `src/services/api/promptCacheBreakDetection.ts`

**两阶段检测**：

```
Phase 1: Pre-call（recordPromptState）
  → 计算 systemHash、toolsHash、cacheControlHash
  → 与前一次比较，记录 pendingChanges

Phase 2: Post-call（checkResponseForCacheBreak）
  → 比较 cacheReadTokens 变化
  → 缓存破裂条件：>5% 下降 && 绝对值 >2000 tokens
  → 分析根因：客户端变化 / 1h TTL 过期 / 服务端变化
  → logEvent('tengu_prompt_cache_break', { ... })
```

### 5.2 TTL 管理与 Latch 机制

```typescript
function should1hCacheTTL(querySource?: QuerySource): boolean {
  // Latch 用户资格（会话开始锁定，防 mid-session 变化）
  let userEligible = getPromptCache1hEligible()
  if (userEligible === null) {
    userEligible = process.env.USER_TYPE === 'ant' ||
      (isClaudeAISubscriber() && !currentLimits.isUsingOverage)
    setPromptCache1hEligible(userEligible)  // 锁定
  }

  // Latch 白名单（同样锁定）
  let allowlist = getPromptCache1hAllowlist()
  if (allowlist === null) {
    allowlist = getFeatureValue('tengu_prompt_cache_1h_config').allowlist ?? []
    setPromptCache1hAllowlist(allowlist)
  }

  // 通配符匹配
  return allowlist.some(p =>
    p.endsWith('*') ? querySource.startsWith(p.slice(0, -1)) : querySource === p
  )
}
```

### 5.3 缓存编辑（Cached Microcompact）

```typescript
// 获取待编辑的缓存删除（一次性消费）
export function consumePendingCacheEdits(): CacheEditsBlock | null {
  const edits = pendingCacheEdits
  pendingCacheEdits = null
  return edits
}

// 固定缓存编辑重放
export function getPinnedCacheEdits(): PinnedCacheEdits[] {
  return cachedMCState?.pinnedEdits ?? []
}

// 通知缓存删除（跳过下次破裂检测）
export function notifyCacheDeletion(querySource, agentId): void {
  state.cacheDeletionsPending = true
}
```

---

## 六、上下文窗口管理

### 6.1 消息处理流水线

```typescript
// 1. 获取压缩边界后的消息
let msgs = [...getMessagesAfterCompactBoundary(messages)]

// 2. 应用工具结果预算
msgs = await applyToolResultBudget(msgs, contentReplacementState)

// 3. Snip 压缩（如果启用 HISTORY_SNIP）
const snipResult = snipCompactIfNeeded(msgs)
msgs = snipResult.messages

// 4. 微级压缩
const mcResult = await microcompact(msgs, toolUseContext, querySource)
msgs = mcResult.messages

// 5. 上下文崩溃（如果启用 CONTEXT_COLLAPSE）
const collapseResult = await contextCollapse.applyCollapsesIfNeeded(msgs)
msgs = collapseResult.messages

// 6. 自动压缩
const { compactionResult } = await autocompact(msgs, ...)
```

### 6.2 自动压缩触发阈值

| 阈值 | 百分比 | 行为 |
|------|--------|------|
| 警告 | 70% | 记录 `calculateTokenWarningState` |
| 压缩 | 85% | 触发 `autocompact()` |
| 阻断 | 95% | 返回错误，保留手动压缩空间 |

---

## 七、多 Provider 切换实现

### 7.1 客户端初始化

**文件**: `src/services/api/client.ts`

```typescript
async function getAnthropicClient({ apiKey, model, ... }) {
  // 1. 构建公共 Headers
  const headers = {
    'x-app': 'cli',
    'User-Agent': getUserAgent(),
    'X-Claude-Code-Session-Id': getSessionId(),
  }

  // 2. OAuth 校验
  await checkAndRefreshOAuthTokenIfNeeded()

  // 3. Provider 路由
  if (CLAUDE_CODE_USE_BEDROCK) {
    return new AnthropicBedrock({ awsRegion, awsAccessKey, ... })
  }
  if (CLAUDE_CODE_USE_FOUNDRY) {
    return new AnthropicFoundry({ azureADTokenProvider, ... })
  }
  if (CLAUDE_CODE_USE_VERTEX) {
    return new AnthropicVertex({ region, googleAuth, ... })
  }

  // 4. 默认第一方 API
  return new Anthropic({
    apiKey: isClaudeAISubscriber() ? null : apiKey,
    authToken: isClaudeAISubscriber() ? oauthToken : undefined,
  })
}
```

### 7.2 区域选择

**Vertex 区域优先级**：
```
1. VERTEX_REGION_CLAUDE_HAIKU_4_5
2. VERTEX_REGION_CLAUDE_3_5_SONNET
3. CLOUD_ML_REGION
4. 默认：us-east5
```

**Bedrock 区域**：
```
1. 主模型：AWS_REGION (us-east-1)
2. Haiku 覆盖：ANTHROPIC_SMALL_FAST_MODEL_AWS_REGION
```

---

## 八、核心算法总结

| 组件 | 算法 | 参数 |
|------|------|------|
| 重试延迟 | 500ms × 2^(n-1) + 25% jitter | 上限 32s |
| 529 处理 | 连续 3 次 → Model fallback | 快速模式冷却 30min |
| AFK 模式 | 心跳 30s，单次最大 5min | 总上限 6h |
| 缓存检测 | 5% 跌幅 && 2000 token 绝对值 | 基于指纹 hash |
| TTL Latch | 会话开始锁定资格/白名单 | 防 mid-session 变化 |
| 自动压缩 | 85% 阈值触发，95% 阻断 | cascading 策略 |
| 媒体限制 | 最多 100 个（图片+文档） | 删除最旧的 |
| Model Fallback | Opus → Sonnet (3 次 529) | Ant 仅内部 |
