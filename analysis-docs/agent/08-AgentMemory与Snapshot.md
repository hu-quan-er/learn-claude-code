# 08 - AgentMemory 与 Snapshot

## 一句话定义

Agent Memory 是给**自定义 agent** 用的持久化记忆系统：每个 agent type 在三种作用域（user / project / local）下有独立的 `MEMORY.md` 文件，agent 启动时被自动加载进 system prompt，agent 可以通过 Write/Edit 工具更新它，下次 spawn 时记忆延续。

这是 Claude Code 里**第三套**记忆体系，和另外两套区别在：

| 记忆体系 | 作用域 | 谁能写 | 自动加载 |
|----------|--------|--------|----------|
| `CLAUDE.md`（项目说明） | 项目级 | 用户 | session 启动时进 system prompt |
| 用户 memory（`~/.claude/memory/`） | 用户级 | 主 agent | session 启动时进 system prompt |
| **Agent Memory** | **per agent type，三层 scope** | **该 agent 自己** | **每次 spawn 进 system prompt** |

## 三层 scope

`src/tools/AgentTool/AgentTool/agentMemory.ts:13`：

```typescript
export type AgentMemoryScope = 'user' | 'project' | 'local'
```

`getAgentMemoryDir`（`agentMemory.ts:52-65`）决定目录：

```typescript
export function getAgentMemoryDir(agentType: string, scope: AgentMemoryScope): string {
  const dirName = sanitizeAgentTypeForPath(agentType)  // 把冒号换成 dash
  switch (scope) {
    case 'project':
      return join(getCwd(), '.claude', 'agent-memory', dirName) + sep
    case 'local':
      return getLocalAgentMemoryDir(dirName)
    case 'user':
      return join(getMemoryBaseDir(), 'agent-memory', dirName) + sep
  }
}
```

| Scope | 目录 | 是否进 VCS | 用途 |
|-------|------|-----------|------|
| `user` | `<memoryBase>/agent-memory/<agentType>/` | 否（用户家目录） | 跨项目复用，agent 学到的"通用经验" |
| `project` | `<cwd>/.claude/agent-memory/<agentType>/` | **是**（团队共享） | 项目内 agent 经验，提交给团队 |
| `local` | `<cwd>/.claude/agent-memory-local/<agentType>/` | 否（gitignore） | 本机本项目，不分享 |

`<memoryBase>` 默认是 `~/.claude/`，可由 `CLAUDE_CODE_MEMORY_DIR` 环境变量覆盖。`local` scope 还支持 `CLAUDE_CODE_REMOTE_MEMORY_DIR` 把"本地"挂载到远程 mount，带项目 namespacing：

```typescript
// agentMemory.ts:29-44
function getLocalAgentMemoryDir(dirName: string): string {
  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    return join(
      process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR,
      'projects',
      sanitizePath(findCanonicalGitRoot(getProjectRoot()) ?? getProjectRoot()),
      'agent-memory-local',
      dirName,
    ) + sep
  }
  return join(getCwd(), '.claude', 'agent-memory-local', dirName) + sep
}
```

适用场景：用户在 cloud workspace / VS Code SSH 这种场景下，希望 local memory 跟着账号走而不是跟工作目录走。

## sanitizeAgentTypeForPath

```typescript
function sanitizeAgentTypeForPath(agentType: string): string {
  return agentType.replace(/:/g, '-')
}
```

把 `:` 换成 `-`。原因：plugin-namespaced 的 agent type 形如 `my-plugin:my-agent`，Windows 文件系统不允许冒号。所以 `my-plugin:reviewer` 对应目录 `my-plugin-reviewer/`。

## 启用条件

agent 定义里写 `memory: 'user' | 'project' | 'local'` 字段：

```markdown
---
name: code-reviewer
description: Reviews PR diffs
tools: Read, Edit, Write
memory: project
---

You are a code reviewer. ...
```

进入 `parseAgentFromJson` / 同等的 markdown frontmatter 解析后：

```typescript
// loadAgentsDir.ts:92
memory: z.enum(['user', 'project', 'local']).optional(),
```

注意：写了 `memory` 字段时，`parseAgentFromJson` **强制注入** Read/Edit/Write 工具到 agent 的 tools 数组：

```typescript
// loadAgentsDir.ts:456-467
if (isAutoMemoryEnabled() && parsed.memory && tools !== undefined) {
  const toolSet = new Set(tools)
  for (const tool of [FILE_WRITE_TOOL_NAME, FILE_EDIT_TOOL_NAME, FILE_READ_TOOL_NAME]) {
    if (!toolSet.has(tool)) {
      tools = [...tools, tool]
    }
  }
}
```

理由：agent 必须能读写 `MEMORY.md`，否则 memory 功能形同虚设。强制注入是为了避免用户写了 `memory: project` 但 `tools: [Bash]` 然后疑惑"为啥 agent 不能写 memory"。

`isAutoMemoryEnabled()` 是 `src/memdir/paths.ts` 里的 feature gate，控制整个 memory 体系是否开启。

## 加载到系统提示

agent spawn 时通过 `getSystemPrompt` 里调 `loadAgentMemoryPrompt`：

```typescript
// loadAgentsDir.ts:482-487
getSystemPrompt: () => {
  if (isAutoMemoryEnabled() && parsed.memory) {
    return (
      systemPrompt + '\n\n' + loadAgentMemoryPrompt(name, parsed.memory)
    )
  }
  return systemPrompt
},
```

`loadAgentMemoryPrompt`（`agentMemory.ts:138-177`）：

```typescript
export function loadAgentMemoryPrompt(agentType: string, scope: AgentMemoryScope): string {
  let scopeNote: string
  switch (scope) {
    case 'user':
      scopeNote = '- Since this memory is user-scope, keep learnings general since they apply across all projects'
      break
    case 'project':
      scopeNote = '- Since this memory is project-scope and shared with your team via version control, tailor your memories to this project'
      break
    case 'local':
      scopeNote = '- Since this memory is local-scope (not checked into version control), tailor your memories to this project and machine'
      break
  }

  const memoryDir = getAgentMemoryDir(agentType, scope)
  void ensureMemoryDirExists(memoryDir)  // fire-and-forget mkdir

  const coworkExtraGuidelines = process.env.CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES
  return buildMemoryPrompt({
    displayName: 'Persistent Agent Memory',
    memoryDir,
    extraGuidelines:
      coworkExtraGuidelines && coworkExtraGuidelines.trim().length > 0
        ? [scopeNote, coworkExtraGuidelines]
        : [scopeNote],
  })
}
```

`buildMemoryPrompt` 在 `src/memdir/memdir.ts`，它干两件事：

1. 读 `<memoryDir>/MEMORY.md` 的内容
2. 读 `<memoryDir>` 下其它 `*.md` 文件作为附加 memory entry
3. 拼装一段系统提示文本，包含 scope 提示 + 内容 + 写 memory 的 guidelines

最终拼到 agent system prompt 末尾，形如：

```text
[原始 agent system prompt]

## Persistent Agent Memory

Memory directory: /Users/.../.claude/agent-memory/code-reviewer/

You have access to persistent memory at the path above. Files in this directory:
- MEMORY.md (auto-loaded)
- specific-pattern.md (auto-loaded)
- ...

Guidelines for writing memories:
- Since this memory is project-scope and shared with your team via version control, tailor your memories to this project
- ...

[MEMORY.md 内容]
```

agent 在 conversation 里看到这段，自然就会"想起"。

`ensureMemoryDirExists` 是 fire-and-forget——注释解释："this runs at agent-spawn time inside a sync getSystemPrompt() callback (called from React render in AgentDetail.tsx, so it cannot be async). The spawned agent won't try to Write until after a full API round-trip, by which time mkdir will have completed."

## isAgentMemoryPath 安全检查

`agentMemory.ts:68-104`：

```typescript
export function isAgentMemoryPath(absolutePath: string): boolean {
  const normalizedPath = normalize(absolutePath)
  const memoryBase = getMemoryBaseDir()

  if (normalizedPath.startsWith(join(memoryBase, 'agent-memory') + sep)) return true
  if (normalizedPath.startsWith(join(getCwd(), '.claude', 'agent-memory') + sep)) return true

  if (process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR) {
    if (
      normalizedPath.includes(sep + 'agent-memory-local' + sep) &&
      normalizedPath.startsWith(join(process.env.CLAUDE_CODE_REMOTE_MEMORY_DIR, 'projects') + sep)
    ) {
      return true
    }
  } else if (normalizedPath.startsWith(join(getCwd(), '.claude', 'agent-memory-local') + sep)) {
    return true
  }
  return false
}
```

`normalize` 是关键的 path traversal 防护。这个函数被权限层调用，判断"这个写入路径是 agent memory（自动允许）还是普通文件（需要权限确认）"。

## Snapshot 同步机制

### 问题背景

`agent-memory-snapshots/` 是一个**项目级 snapshot 目录**，team lead 可以把"团队最佳实践 memory"提交到这里：

```text
<cwd>/.claude/
├── agent-memory/
│   └── code-reviewer/
│       └── MEMORY.md       ← project scope（每个 dev 自己的）
└── agent-memory-snapshots/
    └── code-reviewer/
        ├── snapshot.json   ← { updatedAt: "2026-04-15T..." }
        └── MEMORY.md       ← 团队推荐的 baseline
```

新 dev clone 项目后：

- `agent-memory/code-reviewer/MEMORY.md` 不存在
- snapshot 有 `MEMORY.md`
- 启动时自动 copy snapshot → 本地 memory（initialize）

老 dev 拉取了一个更新的 snapshot：

- 本地有 MEMORY.md（已积累自己的修改）
- snapshot 有更新的版本
- 提示用户"有更新可拉"，等用户决定（prompt-update）

### 实现

`agentMemorySnapshot.ts:98-144` 的 `checkAgentMemorySnapshot`：

```typescript
export async function checkAgentMemorySnapshot(agentType: string, scope: AgentMemoryScope): Promise<{
  action: 'none' | 'initialize' | 'prompt-update'
  snapshotTimestamp?: string
}> {
  const snapshotMeta = await readJsonFile(getSnapshotJsonPath(agentType), snapshotMetaSchema())
  if (!snapshotMeta) return { action: 'none' }  // 项目里没 snapshot

  const localMemDir = getAgentMemoryDir(agentType, scope)

  let hasLocalMemory = false
  try {
    const dirents = await readdir(localMemDir, { withFileTypes: true })
    hasLocalMemory = dirents.some(d => d.isFile() && d.name.endsWith('.md'))
  } catch { /* 目录不存在 */ }

  if (!hasLocalMemory) {
    return { action: 'initialize', snapshotTimestamp: snapshotMeta.updatedAt }
  }

  const syncedMeta = await readJsonFile(getSyncedJsonPath(agentType, scope), syncedMetaSchema())

  if (!syncedMeta || new Date(snapshotMeta.updatedAt) > new Date(syncedMeta.syncedFrom)) {
    return { action: 'prompt-update', snapshotTimestamp: snapshotMeta.updatedAt }
  }

  return { action: 'none' }
}
```

三种 action：

| action | 触发 | 后续 |
|--------|------|------|
| `none` | snapshot 不存在 / 已同步且没更新 | 啥都不做 |
| `initialize` | snapshot 存在但本地无 memory | `initializeFromSnapshot()` 自动 copy |
| `prompt-update` | 本地有 memory，但 snapshot 更新了 | 设置 `agent.pendingSnapshotUpdate`，等用户决定 |

### 三个写动作

```typescript
// agentMemorySnapshot.ts:149-159
export async function initializeFromSnapshot(agentType, scope, snapshotTimestamp): Promise<void> {
  await copySnapshotToLocal(agentType, scope)
  await saveSyncedMeta(agentType, scope, snapshotTimestamp)
}

// agentMemorySnapshot.ts:164-186
export async function replaceFromSnapshot(agentType, scope, snapshotTimestamp): Promise<void> {
  // 先删本地所有 .md（避免 orphan）
  const localMemDir = getAgentMemoryDir(agentType, scope)
  const existing = await readdir(localMemDir, { withFileTypes: true })
  for (const dirent of existing) {
    if (dirent.isFile() && dirent.name.endsWith('.md')) {
      await unlink(join(localMemDir, dirent.name))
    }
  }
  await copySnapshotToLocal(agentType, scope)
  await saveSyncedMeta(agentType, scope, snapshotTimestamp)
}

// agentMemorySnapshot.ts:191-197
export async function markSnapshotSynced(agentType, scope, snapshotTimestamp): Promise<void> {
  await saveSyncedMeta(agentType, scope, snapshotTimestamp)
}
```

| 操作 | 何时调用 | 影响 |
|------|----------|------|
| `initialize` | 首次启动 + snapshot 存在 | copy snapshot 到 local，标记 synced |
| `replace` | 用户选"用 snapshot 覆盖" | 删 local + copy snapshot + 标 synced |
| `markSnapshotSynced` | 用户选"忽略这次更新" | 只更新 .snapshot-synced.json，不动 memory |

### `.snapshot-synced.json`

每个 local memory 目录下有一个 `.snapshot-synced.json`：

```json
{ "syncedFrom": "2026-04-15T10:23:00Z" }
```

记录"我已经看过到这个时间为止的 snapshot"。下次启动比 `snapshotMeta.updatedAt` 大才提示。

### 启动时的自动 init

`loadAgentsDir.ts:262-294` 的 `initializeAgentMemorySnapshots`：

```typescript
async function initializeAgentMemorySnapshots(agents: CustomAgentDefinition[]): Promise<void> {
  await Promise.all(
    agents.map(async agent => {
      if (agent.memory !== 'user') return  // 只 user scope 走自动 init
      const result = await checkAgentMemorySnapshot(agent.agentType, agent.memory)
      switch (result.action) {
        case 'initialize':
          await initializeFromSnapshot(agent.agentType, agent.memory, result.snapshotTimestamp!)
          break
        case 'prompt-update':
          agent.pendingSnapshotUpdate = { snapshotTimestamp: result.snapshotTimestamp! }
          break
      }
    }),
  )
}
```

注意：**只有 `memory: 'user'` 的 agent 走自动 init**。project / local scope 不自动 init，因为：

- project：项目自己的 git 已经管 memory 文件了，snapshot 是冗余的
- local：local 是机器特有的，从 snapshot init 没意义

只有 user scope 才需要"跨项目同步 baseline"——所以 snapshot 也只对 user scope 有意义。

`feature('AGENT_MEMORY_SNAPSHOT')` 控制整个 snapshot 系统是否启用。

## 完整的 agent memory 生命周期

```text
[首次：agent 定义里 memory: 'user']
1. session 启动 → getAgentDefinitionsWithOverrides
2. initializeAgentMemorySnapshots：
   - 检查 ~/.claude/agent-memory/<agentType>/
   - 不存在 + 项目 snapshot 存在 → copy snapshot → 写 .snapshot-synced.json
3. agent spawn 时 getSystemPrompt → loadAgentMemoryPrompt(...)
   - 读 MEMORY.md + 其它 .md
   - 拼到 system prompt 末尾
4. agent 在 conversation 中 Write/Edit MEMORY.md → 文件被改
5. agent 结束，下次 spawn 重复 3
6. session 启动时如果 snapshot 更新了 → pendingSnapshotUpdate 标记 → UI 提示用户

[后续：用户选择行为]
- "keep mine" → markSnapshotSynced：只更新时间戳
- "use snapshot" → replaceFromSnapshot：删本地 .md，重新 copy snapshot
- 不操作 → 下次启动还会提示
```

## 几个易踩点

1. **memory 字段必须显式声明**：不写就是 undefined，agent 没 memory。
2. **写 memory 字段会强制注入 Read/Edit/Write**：即使你 `tools: [Bash]` 也会被加这三个工具，因为 memory 没工具就用不起来。
3. **snapshot 系统只支持 user scope**：project/local scope 没有 snapshot 概念。
4. **snapshot 是项目级的 (`<cwd>/.claude/agent-memory-snapshots/`)，但同步的是 user-scope memory**：这是"项目向所有 dev 推送通用经验"的设计——项目里 commit 一份 snapshot，每个 dev 的 ~/.claude/agent-memory/ 会被自动 init。
5. **memory 内容直接出现在 API request 里**：每次 spawn 重新加载，token cost 累加。`MEMORY.md` 写太长会拖累子代理性能。
6. **复用 `buildMemoryPrompt`**：和主代理用户 memory 同一套渲染——所以 agent memory 的 instructive guidelines 也和主代理一致（"keep entries focused" 等）。
7. **`ensureMemoryDirExists` 是 fire-and-forget**：sync 上下文限制不能 await mkdir。极端情况下 mkdir 还没完成 agent 就尝试 Write——`FileWriteTool` 自己会 mkdir 父目录兜底。
8. **`isAgentMemoryPath` 用 `normalize` 防 path traversal**：写 memory 的权限策略依赖这个判断，traversal 漏洞会让恶意 agent 写到 memory 目录之外。
