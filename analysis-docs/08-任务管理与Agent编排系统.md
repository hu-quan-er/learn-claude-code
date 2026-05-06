# 08 - 任务管理与 Agent 编排系统

---

## 一、任务框架完整生命周期

### 1.1 任务状态类型

**文件**: `src/tasks/types.ts`

```typescript
type TaskState =
  | LocalShellTaskState         // 本地 Shell/Bash 任务
  | LocalAgentTaskState         // 本地代理任务
  | RemoteAgentTaskState        // 远程 CCR 代理任务
  | InProcessTeammateTaskState  // 进程内团队成员任务
  | LocalWorkflowTaskState      // 本地工作流任务
  | MonitorMcpTaskState         // MCP 监控任务
  | DreamTaskState              // 记忆梦想提取任务
```

### 1.2 任务生命周期阶段

```
创建阶段 (registerTask)
  → registerTask(taskState, setAppState)
  → 发出 task_started SDK 事件
  → 保存初始元数据（ID、描述、类型、开始时间）

执行阶段 (running)
  → 持续监控和更新进度
  → 写入输出到磁盘（TaskOutput 类）
  → 发出 task_progress SDK 事件

进度跟踪 (AgentProgress)
  → toolUseCount: 工具使用计数
  → tokenCount: Token 消耗统计
  → lastActivity / recentActivities: 最近 5 个活动
  → summary: 进度摘要

完成/失败 (terminal status)
  → 状态转入 completed / failed / killed
  → 设置 endTime 时间戳
  → enqueueAgentNotification() / enqueueShellNotification()
  → 设置 notified=true 防止重复通知
  → evictTaskOutput() 清理磁盘缓存
```

### 1.3 任务轮询与清理

- `pollTasks()` — 每 1 秒轮询运行中的任务
- `generateTaskAttachments()` — 生成任务状态变化的附件
- `applyTaskOffsetsAndEvictions()` — 输出偏移量更新和任务清理
- 终端任务在标记为 `notified` 后被清理

---

## 二、Agent 子代理创建与管理

### 2.1 子代理创建参数

**文件**: `src/tools/AgentTool/AgentTool.tsx`

```typescript
// 基础参数
{
  description: string,            // 3-5 字任务描述
  prompt: string,                 // 完整任务提示
  subagent_type?: string,         // 代理类型
  model?: 'sonnet' | 'opus' | 'haiku',
  run_in_background?: boolean,
}

// 多代理参数
{
  name?: string,                  // 团队成员名称（可寻址）
  team_name?: string,             // 团队名称
  mode?: PermissionMode,          // 权限模式
  isolation?: 'worktree' | 'remote',
  cwd?: string,                   // 工作目录覆盖
}
```

### 2.2 隔离模式

| 模式 | 说明 | 特点 |
|------|------|------|
| **无隔离**（默认） | 继承父代理工作目录 | 最快、共享文件系统 |
| **Worktree** | 独立 Git 工作树 | 隔离副本、同仓库、`createAgentWorktree()` 管理 |
| **Remote** | 远程 CCR 环境 | 网络隔离、`teleportToRemote()`、总是后台运行 |

### 2.3 模型继承与覆盖

```
模型选择优先级：
1. 显式 model 参数
2. Agent 定义中的 model
3. 父代理模型（继承）
4. 全局默认模型
```

### 2.4 权限传递

**权限模式**：
- `'bubble'` — 权限提示上浮到父代理
- `'plan'` — 需要计划批准
- `'manager'` — 继承父权限

**工具访问**：
- `allowedTools` 显式列表 — 子代理只能使用列出的工具
- MCP 服务器 — 继承父代理连接，Agent 完成时清理新连接

### 2.5 Fork 子代理模式

当 `FORK_SUBAGENT` 功能启用时：

```typescript
const FORK_AGENT = {
  agentType: 'fork',
  tools: ['*'],              // 继承父工具池
  model: 'inherit',          // 继承父模型
  permissionMode: 'bubble',  // 权限上浮
}
```

Fork child 收到：
- 完整的父对话历史
- 父系统提示（字节完全相同，用于提示缓存命中）
- 所有工具使用块作为 tool_result（缓存共享）
- 每个 child 独有的指令文本块

**递归防护**：检测 fork 样板 → `throw Error('Fork 不能在 fork 子进程中使用')`

---

## 三、内置 Agent 类型

### 3.1 Agent 定义结构

```typescript
type AgentDefinition = {
  agentType: string
  whenToUse: string              // UI 用途描述
  tools?: string[]               // ['*'] 或具体名称
  model?: 'inherit' | ModelAlias
  maxTurns?: number
  source: 'built-in' | 'plugin' | 'userSettings'
  baseDir: string
  mcpServers?: (string | Record<string, McpConfig>)[]
  requiredMcpServers?: string[]
  getSystemPrompt(): string
  color?: string                 // UI 着色
  background?: boolean           // 强制后台运行
}
```

### 3.2 内置 Agent 列表

| Agent | 用途 | 工具访问 |
|-------|------|----------|
| **General-Purpose** | 通用 — 研究、搜索、多步任务 | 所有工具 (`*`) |
| **Explore** | 代码库探索和分析 | 只读工具 |
| **Plan** | 复杂实现的规划 | 只读工具 |
| **Claude Code Guide** | 指导用户使用 Claude Code | 只读 + WebFetch |
| **Statusline Setup** | 配置 statusline 集成 | Read + Edit |
| **Fork** | 隐式 fork 子代理（实验） | 继承父工具 |

---

## 四、协调器模式工作流

### 4.1 启用条件

```typescript
function isCoordinatorMode(): boolean {
  return feature('COORDINATOR_MODE') &&
         isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
}
```

### 4.2 协调器角色

协调器从系统提示中获得编排能力：
- **角色**：编排者，不是独立执行者
- **工具**：`Agent`（生成 worker）、`SendMessage`（继续 worker）、`TaskStop`（停止 worker）

### 4.3 任务分解策略

```
研究阶段    → Workers 并行执行（多个角度调查）
综合阶段    → 协调器理解问题、制定规范
实现阶段    → Workers 顺序执行（避免文件冲突）
验证阶段    → Workers 并行或顺序（取决于独立性）
```

**并发管理**：
- **只读任务**（研究）— 自由并行
- **写入任务**（实现）— 相同文件集一次一个
- **验证** — 有时与实现并行（不同文件区域）

### 4.4 Worker 结果格式

```xml
<task-notification>
  <task-id>agent-a1b</task-id>
  <status>completed|failed|killed</status>
  <summary>Agent "Description" completed</summary>
  <result>Agent's final text response</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

### 4.5 继续 vs 生成新 Worker 决策

| 继续 (SendMessage) | 生成新 Worker (Agent) |
|--------------------|-----------------------|
| 研究完全覆盖实现文件 | 研究宽泛但实现窄 |
| Worker 已有目标文件上下文 | 避免拖带探索噪声 |
| 更正失败或扩展近期工作 | 验证不同 Worker 的代码（新鲜视角） |

---

## 五、远程会话管理（CCR 模式）

### 5.1 RemoteAgentTask 状态

```typescript
type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  remoteTaskType: 'remote-agent' | 'ultraplan' | 'ultrareview' | 'autofix-pr'
  sessionId: string
  command: string
  title: string
  todoList: TodoList
  log: SDKMessage[]
  pollStartedAt: number
  reviewProgress?: {
    stage: 'finding' | 'verifying' | 'synthesizing'
    bugsFound: number
    bugsVerified: number
    bugsRefuted: number
  }
}
```

### 5.2 WebSocket 架构

**RemoteSessionManager**:
```typescript
class RemoteSessionManager {
  connect(): void                           // 连接远程会话
  sendMessage(content): Promise<boolean>    // 发送用户消息
  respondToPermissionRequest()              // 响应权限请求
  cancelSession(): void                     // 发送中断信号
  isConnected(): boolean
}
```

**消息流**：
```
1. HTTP POST 初始化会话
2. WebSocket 订阅（双向消息通道）
3. HTTP POST 发送用户消息
4. WebSocket 接收 Assistant 消息、权限请求
5. WebSocket 回复权限决定
```

### 5.3 权限请求气泡流程

```
Remote CCR:
  "可以使用工具 X 吗？"
  → control_request(can_use_tool)

本地 Bridge:
  → onPermissionRequest 回调
  → 显示权限提示给用户

用户批准/拒绝
  → respondToPermissionRequest()
  → control_response

Remote CCR:
  → 继续或停止工具执行
```

---

## 六、Bridge 通信层

### 6.1 消息协议

**进入消息处理**：
```typescript
handleIngressMessage(
  data: string,                    // WebSocket 数据
  recentPostedUUIDs,              // 回声检测
  recentInboundUUIDs,             // 重复交付检测
  onInboundMessage,               // SDK 消息回调
  onPermissionResponse,           // 权限响应回调
  onControlRequest                // 控制请求回调
)
```

**消息去重**：
- UUID 跟踪：避免处理自己发送的回声
- 序列号继续：WebSocket 崩溃恢复
- 有界 UUID 集：内存有限

### 6.2 Server 控制请求

```typescript
type ServerControlRequestHandlers = {
  transport: ReplBridgeTransport | null
  sessionId: string
  outboundOnly?: boolean                // 只读模式
  onInterrupt?: () => void              // 用户中断
  onSetModel?: (model: string) => void  // 模型变更
  onSetPermissionMode?: (mode) => void  // 权限模式变更
}
```

---

## 七、Swarm 多代理后端

### 7.1 后端架构

```typescript
type TeammateExecutor = {
  type: 'in-process' | 'tmux' | 'iterm2'
  spawn(config): Promise<TeammateSpawnResult>
  sendMessage(agentId, message): Promise<void>
  terminate(agentId, reason?): Promise<boolean>
}
```

**后端检测优先级**：`in-process > tmux > iTerm2`

### 7.2 进程内后端

```typescript
class InProcessBackend implements TeammateExecutor {
  async spawn(config) {
    // 1. 创建 TeammateContext
    // 2. 创建独立 AbortController
    // 3. 在 AppState 中注册
    // 4. 启动代理执行循环
  }

  async sendMessage(agentId, message) {
    // 写入文件基础邮箱
  }

  async terminate(agentId, reason) {
    // 发送 shutdown 请求消息
    // 设置 shutdownRequested 标志
  }
}
```

### 7.3 Tmux 后端

使用 tmux 创建独立的 pane，代理在专用 terminal 中运行，通过 IPC 邮箱通信。

### 7.4 团队上下文

```typescript
type TeamContext = {
  name: string
  members: TeamMember[]
  leadAgentId: string
  taskMap: Record<agentId, taskId>
}
```

**管理机制**：
- 文件基础 team manifest
- AsyncLocalStorage 隔离 teammate 上下文
- 邮箱系统用于进程间通信
- 权限与 leader 同步

---

## 八、Buddy 系统（伴侣角色）

### 8.1 Buddy 生成

**文件**: `src/buddy/companion.ts`

```typescript
function roll(userId: string): Roll {
  // 确定性生成：基于 userId + salt，Mulberry32 PRNG
  return {
    bones: {
      rarity: 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary',
      species: 'duck' | 'fox' | ...,
      eye: 'round' | 'mysterious' | ...,
      hat: 'none' | 'beret' | ...,
      shiny: boolean,  // 1% 概率闪闪发光
      stats: { accuracy, efficiency, friendliness }
    }
  }
}
```

### 8.2 Buddy 组件

- `CompanionSprite.tsx` — 动画精灵渲染（表情变化、交互反应）
- `useBuddyNotification.tsx` — 基于任务事件的通知气泡

---

## 九、DreamTask — 自动内存提取

```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  filesTouched: string[]
  turns: DreamTurn[]
  abortController?: AbortController
  priorMtime: number
}
```

Dream 任务特点：
- 后台自动运行
- 合并并总结会话历史
- 提取关键见解到记忆系统
- 仅显示 UI，不发送模型面向通知

---

## 十、本地 Shell 任务

### 10.1 Shell 任务状态

```typescript
type LocalShellTaskState = TaskStateBase & {
  type: 'local_bash'
  command: string
  shellCommand: ShellCommand
  isBackgrounded: boolean
  result?: { code: number; interrupted: boolean }
}
```

### 10.2 Stall 检测

```
STALL_THRESHOLD_MS = 45 秒无输出增长
→ 检查输出尾部是否包含提示模式 (y/n), [Y/n], 问题
→ 通知用户可能卡在交互式提示
```

---

## 十一、架构设计原则总结

| 原则 | 实现 |
|------|------|
| **统一任务框架** | 所有后台工作注册为 Task，共同生命周期 |
| **隔离执行** | fork / worktree / remote 三种隔离模式 |
| **异步编排** | 协调器支持任务分解、并行执行、结果合成 |
| **消息驱动** | XML task-notification + SDK 事件异步通信 |
| **权限气泡** | 权限决定向上传播，用户始终掌控 |
| **进度透明** | 实时进度事件和输出流 |
| **优雅降级** | Bridge 单向模式、Swarm 自动检测最佳后端 |
| **内存管理** | 积极的任务清理、输出截断、文件基础持久化 |
