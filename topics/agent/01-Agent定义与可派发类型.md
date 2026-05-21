# 01 - Agent 定义与可派发类型

## 三类来源

Claude Code 把所有 agent 统一表示成 `AgentDefinition`（`loadAgentsDir.ts:106-165`），按来源分三层结构、五种 source 标记：

```typescript
export type AgentDefinition =
  | BuiltInAgentDefinition       // source: 'built-in'
  | CustomAgentDefinition        // source: 'userSettings' | 'projectSettings' | 'policySettings' | 'flagSettings'
  | PluginAgentDefinition        // source: 'plugin'
```

加载入口是 `getAgentDefinitionsWithOverrides(cwd)`（`loadAgentsDir.ts:296`），返回 `{ activeAgents, allAgents, failedFiles }`。`activeAgents` 经过 `getActiveAgentsFromList()` 按以下优先级**覆盖式合并**（同名 agentType 后者覆盖前者）：

```text
1. built-in        ← 内置 6 个
2. plugin          ← .claude/plugins/*/agents/*.md
3. userSettings    ← ~/.claude/agents/*.md
4. projectSettings ← <cwd>/.claude/agents/*.md
5. flagSettings    ← --agents CLI flag 注入
6. policySettings  ← MDM policy 注入（最高优先级）
```

也就是说：项目级可以覆盖用户级，policy 可以覆盖一切。同名是按 `agentType` 字符串匹配的。

## AgentDefinition 字段清单

所有 agent 共享的字段（`BaseAgentDefinition`，`loadAgentsDir.ts:106-133`）：

| 字段 | 类型 | 含义 |
|------|------|------|
| `agentType` | `string` | 唯一标识，对应 `Agent({ subagent_type: ... })` |
| `whenToUse` | `string` | 给父代理看的描述，进入 Agent 工具的 prompt 第二段 |
| `tools` | `string[]?` | 工具白名单（含 `'*'` 通配） |
| `disallowedTools` | `string[]?` | 工具黑名单（与 `tools` 同时存在时取交集） |
| `skills` | `string[]?` | 启动时预加载的 skill 名 |
| `mcpServers` | `AgentMcpServerSpec[]?` | 此 agent 专属的 MCP servers |
| `hooks` | `HooksSettings?` | session 级 hooks，agent 启动时注册 |
| `color` | `AgentColorName?` | UI 颜色（teal/orange/red/...） |
| `model` | `string?` | 模型 alias（`sonnet`/`opus`/`haiku`/`inherit`）或完整 ID |
| `effort` | `EffortValue?` | thinking 强度（low/medium/high 或数字 token 预算） |
| `permissionMode` | `PermissionMode?` | `default` / `acceptEdits` / `bypassPermissions` / `plan` / `dontAsk` |
| `maxTurns` | `number?` | agent 内部循环最大轮数 |
| `background` | `boolean?` | spawn 时强制后台运行 |
| `initialPrompt` | `string?` | 拼到首轮 user message 前的额外指令（支持 slash command） |
| `memory` | `'user'/'project'/'local'?` | 持久记忆作用域 |
| `isolation` | `'worktree'/'remote'?` | 创建时默认隔离方式（remote 仅 ant） |
| `omitClaudeMd` | `boolean?` | 跳过 CLAUDE.md 注入（read-only agent 默认 true，省 token） |
| `requiredMcpServers` | `string[]?` | 必须存在的 MCP server 名通配，否则此 agent 不可见 |
| `criticalSystemReminder_EXPERIMENTAL` | `string?` | 每个 user turn 都重新注入的短提醒 |

`BuiltInAgentDefinition` 多一个 `getSystemPrompt(params)`（带 `toolUseContext`），允许动态拼装；`CustomAgentDefinition` 是 `getSystemPrompt(): string`（闭包返回 markdown body）。

## 自定义 agent 的两种格式

### Markdown 格式（最常见）

`.claude/agents/<name>.md`：

```markdown
---
name: code-reviewer
description: Reviews PR diffs for correctness, style, and security
tools: Read, Grep, Bash, Edit
model: sonnet
permissionMode: default
---

You are a senior code reviewer. Read the diff and report:
1. Correctness issues
2. Style violations
3. Security risks

Be concrete. Cite file paths and line numbers.
```

frontmatter 字段经 `markdownConfigLoader.ts` 解析，body 成为 `systemPrompt`。文件名（去 `.md` 后缀）默认是 `agentType`，但 frontmatter 里的 `name` 字段会覆盖。

### JSON 格式（flag / policy 用）

通过 `--agents '{"name1": {...}, "name2": {...}}'` 或 policy settings 注入时走 `parseAgentFromJson()`（`loadAgentsDir.ts:445`）。Zod schema 是 `AgentJsonSchema`：

```typescript
{
  description: string,
  tools: string[]?,
  disallowedTools: string[]?,
  prompt: string,          // 完整的 system prompt 字符串
  model: string?,          // 'inherit' 关键字会被规范化
  effort, permissionMode, mcpServers, hooks, maxTurns, skills, initialPrompt,
  memory: 'user'|'project'|'local'?,
  background: boolean?,
  isolation: 'worktree' (ant 还能选 'remote')
}
```

## 6 个内置 agent

完整定义在 `src/tools/AgentTool/built-in/`。

### 注册与 feature gate

`builtInAgents.ts:22-72` 是入口：

```typescript
const agents: AgentDefinition[] = [
  GENERAL_PURPOSE_AGENT,
  STATUSLINE_SETUP_AGENT,
]

if (areExplorePlanAgentsEnabled()) {           // tengu_amber_stoat
  agents.push(EXPLORE_AGENT, PLAN_AGENT)
}

if (isNonSdkEntrypoint) {                      // 非 SDK 入口才注入
  agents.push(CLAUDE_CODE_GUIDE_AGENT)
}

if (feature('VERIFICATION_AGENT') && tengu_hive_evidence) {
  agents.push(VERIFICATION_AGENT)
}
```

环境变量也能干预：

- `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1`（仅非交互式）：清空所有内置 agent，SDK 用户拿到"白板"。
- `CLAUDE_CODE_COORDINATOR_MODE=1`：走 coordinator 路径，注入一组 worker agents 而不是上面这套。
- `CLAUDE_CODE_SIMPLE=1`：跳过自定义 agent 加载，只用内置。
- `CLAUDE_CODE_ENTRYPOINT=sdk-ts/sdk-py/sdk-cli`：抑制 `claude-code-guide`。

### 6 个内置 agent 对比

| agentType | source | model | 工具策略 | omitClaudeMd | 其他 |
|-----------|--------|-------|----------|--------------|------|
| `general-purpose` | built-in | 默认 `'inherit'` | `tools: ['*']` | false | 通用研究 / 多步骤任务 |
| `Explore` | built-in（feature gated） | ant: `'inherit'`，外部: `'haiku'` | `disallowedTools: [Agent, ExitPlanMode, Edit, Write, NotebookEdit]` | true | 快速只读搜索 |
| `Plan` | built-in（feature gated） | `'inherit'` | 同 Explore 的 disallowedTools，且 `tools` 继承 Explore | true | 设计实施方案，必须 Read-only |
| `verification` | built-in（feature + GB gate） | `'inherit'` | 同 Explore 的 disallowedTools | false | 用 PASS/FAIL/PARTIAL 给出验证判决 |
| `claude-code-guide` | built-in（仅非 SDK） | `'haiku'` | `[Bash 或 Glob/Grep, Read, WebFetch, WebSearch]` | 默认 | 回答 Claude Code / SDK / API 文档问题，`permissionMode: 'dontAsk'` |
| `statusline-setup` | built-in | `'sonnet'` | `tools: ['Read', 'Edit']` | 默认 | 一次性工具，配置 statusline，`color: 'orange'` |

### 几个内置 agent 的关键设计

**Explore：模型选择有 ant/外部分支**

```typescript
// exploreAgent.ts:78
model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
```

注释解释：ant 用户用主模型保证质量；外部用户用 haiku 换速度。这是源码里少数显式按 `USER_TYPE` 分流的 agent。Explore 还在系统提示里硬钉 "READ-ONLY MODE — NO FILE MODIFICATIONS"，是 prompt + disallowedTools 双层保险。

**Verification：要求结构化输出 + 抗"自我安慰"**

```typescript
// verificationAgent.ts:150
criticalSystemReminder_EXPERIMENTAL:
  'CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or create files IN THE PROJECT DIRECTORY (tmp is allowed for ephemeral test scripts). You MUST end with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL.',
```

每次 user turn 都会再注入这条 reminder（这是 `criticalSystemReminder_EXPERIMENTAL` 字段唯一被使用的地方）。Verification agent 的 prompt 还有专门的"反 rationalization"段落，列出模型容易找的借口（"代码读起来对的"/"测试已经过了"），是 prompt engineering 上比较有意思的样本。

**Claude-code-guide：动态拼装 prompt**

它的 `getSystemPrompt` 不是静态字符串，而是接 `toolUseContext` 拼装的：在 base prompt 后追加用户当前的 `commands` / `customAgents` / `mcpClients` / `pluginCommands` / `settings.json`，让 guide agent 拿到完整的"用户配置切面"。这是 built-in agent 用 `getSystemPrompt(params)` 而非 `() => string` 的少数案例。

**statusline-setup：固定 sonnet 模型**

唯一一个**不 inherit、不 haiku、而是硬指 `'sonnet'`** 的内置 agent。原因可以推测：要解析 PS1 + 写 JSON 配置，要求语义理解 + 文本生成质量，不能用 haiku。

**Plan：复用 Explore 的 tools**

```typescript
// planAgent.ts:85
tools: EXPLORE_AGENT.tools,
```

直接引用 Explore 的工具配置，避免重复定义。Plan 的"必须 Read-only"约束和 Explore 一样靠 prompt + disallowedTools 双保险，但它额外要求输出固定格式的"Critical Files for Implementation"段落。

## 可派发类型的两层过滤

在父代理实际能"派发出哪些类型"之前，还要经过两层过滤（`AgentTool.tsx:217-219`）：

```typescript
// 1. MCP 要求过滤：requiredMcpServers 必须全部命中当前 session 可用 MCP server
const agentsWithMcpRequirementsMet = filterAgentsByMcpRequirements(agents, mcpServersWithTools);

// 2. 权限规则过滤：permissions deny 规则
const filteredAgents = filterDeniedAgents(agentsWithMcpRequirementsMet, toolPermissionContext, AGENT_TOOL_NAME);
```

`requiredMcpServers` 这个机制让 agent 定义可以声明 "我依赖 `playwright` / `slack` / `linear` 才有意义"。当对应 MCP server 没配置时，这个 agent 不会出现在父代理可见的列表里——既减少误派，也避免子代理启动后才发现工具缺失。

`filterDeniedAgents` 则按用户配置的 deny 规则筛掉 "禁止 spawn 的 agent 类型"。源码里管这个叫"agent 维度的权限"，和工具维度的权限正交。

## 工具白/黑名单的最终生效逻辑

`prompt.ts:15-37` 的 `getToolsDescription` 给出了 effective tools 的描述规则（也对应 runtime 行为）：

| 配置 | effective tools 描述 |
|------|----------------------|
| 仅有 `tools` | 完全等于 `tools` 数组 |
| 仅有 `disallowedTools` | 父代理所有工具去掉 `disallowedTools` |
| 两者都有 | `tools` 数组中去掉 `disallowedTools` 后剩余项 |
| 都没有 | 父代理所有工具 |

要注意：`tools: ['*']` 不是字面意义"所有"，而是在 `parseAgentToolsFromFrontmatter` 中被翻译成"父代理可见工具的全集"。具体在 07 篇展开。

## 一个最小自定义 agent 例子

```markdown
---
name: pr-summary
description: Summarize a PR diff for changelog
tools: Bash, Read
model: haiku
---

You are given a PR diff. Output:
1. One-line summary (under 80 chars)
2. Bullet list of user-facing changes
3. Any breaking changes flagged

Use `git diff` via Bash to fetch the diff if needed. Don't speculate beyond what the diff shows.
```

放到 `.claude/agents/pr-summary.md` 后，父代理在 prompt 的 agent 列表里就会看到：

```text
- pr-summary: Summarize a PR diff for changelog (Tools: Bash, Read)
```

调用形态：

```json
{
  "name": "Agent",
  "input": {
    "subagent_type": "pr-summary",
    "description": "Summarize current branch PR",
    "prompt": "Summarize the diff from main..HEAD for the changelog. Include any DB migration warnings."
  }
}
```

## 常见困惑

1. **同一个 agentType 出现两次怎么办？** —— 由 `getActiveAgentsFromList()` 决定。built-in < plugin < user < project < flag < policy，后者覆盖前者。所以项目级 `.claude/agents/Explore.md` 可以覆盖内置 Explore。
2. **agent 定义里的 `model: 'sonnet'` 真的就用 sonnet 吗？** —— 不一定。父代理可以通过 `Agent({ model: 'haiku', ... })` 覆盖；`CLAUDE_CODE_SUBAGENT_MODEL` 环境变量再优先于它。完整优先级见 04 篇。
3. **`tools: ['*']` 和不写 `tools` 字段有什么区别？** —— 行为上等价于"用父代理所有工具"；但写 `['*']` 是显式表态"我故意要全集"，避免被误以为忘记配置。
4. **`requiredMcpServers` 是 hard requirement 还是 soft？** —— hard。MCP server 不可用时 agent 直接从父代理可见列表里消失，不会"先 spawn 再失败"。
5. **policy agent 用什么场景？** —— 企业 MDM 注入。用 `settings.json` 的 `agents` 字段，在 policySettings 层 inject，无法被用户/项目覆盖。
