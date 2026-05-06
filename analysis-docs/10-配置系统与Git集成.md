# 10 - 配置系统与 Git 集成

---

## 一、配置系统架构概述

### 1.1 六级设置优先级

```
优先级（从高到低）：

1. policySettings  — 企业托管设置
   ├── 远程托管设置 API（最高）
   ├── 管理员 MDM（HKLM/macOS plist）
   ├── managed-settings.json + managed-settings.d/*.json
   └── HKCU 注册表（Windows 用户可写）

2. flagSettings    — CLI 标志设置（--settings 和 SDK 内联）

3. localSettings   — 项目本地设置（.claude/settings.local.json，gitignored）

4. projectSettings — 共享项目设置（.claude/settings.json，版本控制）

5. userSettings    — 全局用户设置（~/.claude/settings.json）

6. pluginSettings  — 插件提供的基础设置（最低）
```

### 1.2 设置文件位置

| 来源 | 位置 | 作用域 |
|------|------|--------|
| **userSettings** | `~/.claude/settings.json` 或 `~/.claude/cowork_settings.json` | 全局/用户 |
| **projectSettings** | `.claude/settings.json` | 项目级（共享） |
| **localSettings** | `.claude/settings.local.json` | 项目级（本地，自动 gitignore） |
| **policySettings** | `~/.config/managed-settings.json` 或远程 API | 企业级 |
| **flagSettings** | CLI `--settings` 标志 | 会话级 |

### 1.3 核心文件职责

| 文件 | 职责 |
|------|------|
| `src/utils/settings/settings.ts` | 主设置加载、合并、缓存逻辑 |
| `src/utils/settings/types.ts` | Zod Schema 定义（SettingsSchema） |
| `src/utils/settings/settingsCache.ts` | 会话级与按来源缓存 |
| `src/utils/settings/validation.ts` | 两阶段验证（预过滤 + Zod） |
| `src/utils/settings/permissionValidation.ts` | 权限规则验证 |
| `src/utils/settings/mdm/settings.ts` | 平台特定 MDM 策略处理 |

---

## 二、设置 Schema 核心字段

### 2.1 认证与凭证

```typescript
{
  apiKeyHelper?: string                  // 输出认证值的脚本
  awsCredentialExport?: string           // AWS 凭证导出脚本
  gcpAuthRefresh?: string                // GCP 认证刷新命令
  xaaIdp?: {                             // OIDC IdP 配置
    issuer: string
    clientId: string
    callbackPort: number
  }
}
```

### 2.2 权限与安全

```typescript
{
  permissions?: {
    allow?: PermissionRule[]             // 允许列表
    deny?: PermissionRule[]              // 拒绝列表
    ask?: PermissionRule[]               // 始终询问列表
    defaultMode?: 'allow' | 'deny' | 'ask'
    additionalDirectories?: string[]     // 扩展权限范围
  }
}
```

**权限规则格式**：
```
Bash(npm run:*)          // 旧版前缀匹配
Bash(npm install)        // 精确命令
FileEdit(/src/**.js)     // 文件模式（glob）
Bash(git *)              // 通配符模式
mcp__web__search         // MCP 服务器工具访问
```

### 2.3 Git 与归属

```typescript
{
  attribution?: {
    commit?: string                     // 自定义提交归属
    pr?: string                         // 自定义 PR 归属
  }
  includeCoAuthoredBy?: boolean         // 废弃：使用 attribution
  includeGitInstructions?: boolean      // 包含 git 工作流说明（默认: true）

  worktree?: {
    symlinkDirectories?: string[]       // 要符号链接的目录
    sparsePaths?: string[]              // git sparse-checkout cone 模式
  }
}
```

### 2.4 MCP 服务器

```typescript
{
  allowedMcpServers?: AllowedMcpServerEntry[]   // 企业允许列表
  deniedMcpServers?: DeniedMcpServerEntry[]     // 企业拒绝列表
  enableAllProjectMcpServers?: boolean           // 自动批准项目服务器
  enabledMcpjsonServers?: string[]               // 批准的 .mcp.json 服务器
}
```

### 2.5 钩子与自动化

```typescript
{
  hooks?: HooksSettings                  // 工具操作的自定义命令
  allowManagedHooksOnly?: boolean        // 仅使用托管钩子
  allowedHttpHookUrls?: string[]         // HTTP 钩子目标允许列表
  httpHookAllowedEnvVars?: string[]      // HTTP 钩子可用环境变量
}
```

### 2.6 模型与行为

```typescript
{
  model?: string                         // 默认模型覆盖
  availableModels?: string[]             // 企业模型允许列表
  modelOverrides?: Record<string, string> // 模型 ID 映射
  alwaysThinkingEnabled?: boolean        // 扩展思考（默认: true）
  effortLevel?: 'low' | 'medium' | 'high' | 'max'
  fastMode?: boolean
  language?: string                      // 首选语言
  outputStyle?: string                   // 响应风格定制
}
```

### 2.7 UI 与显示

```typescript
{
  statusLine?: { type: 'command', command: string, padding?: number }
  spinnerTipsEnabled?: boolean
  spinnerVerbs?: { mode: 'append' | 'replace', verbs: string[] }
  syntaxHighlightingDisabled?: boolean
  terminalTitleFromRename?: boolean
  feedbackSurveyRate?: number            // 调查概率（0-1）
}
```

---

## 三、设置加载与合并

### 3.1 加载流程（getInitialSettings）

```
1. 会话启动
   → getAllowedSettingSources() 从 CLI 标志获取
   → 'policySettings' 和 'flagSettings' 始终包含

2. 按来源加载（按优先级顺序）
   → Plugin → User → Project → Local → Flag → Policy

3. 策略设置解析（"首个来源获胜"）
   → 远程托管设置 API（最高）
   → 管理员 MDM（HKLM/macOS plist）
   → managed-settings.json + managed-settings.d/*.json
   → HKCU 注册表

4. 深度合并（自定义规则）
   → 数组：连接 + 去重（uniq()）
   → 对象：递归合并
   → undefined 值触发从映射中删除

5. 验证
   → 先过滤权限规则（单个坏规则不影响整体）
   → Zod safeParse() 验证 Schema
   → 错误收集但不阻止加载

6. 缓存
   → 会话缓存（按来源 + 合并结果）
   → 解析缓存（按文件路径，LRU）
   → 设置写入时失效缓存
```

### 3.2 合并策略

```typescript
function settingsMergeCustomizer(objValue, srcValue) {
  if (Array.isArray(objValue) && Array.isArray(srcValue)) {
    return uniq([...objValue, ...srcValue])  // 去重连接
  }
  return undefined  // 让 lodash 处理默认合并
}
```

**合并示例**：
```json
// 用户设置
{ "permissions": { "allow": ["FileEdit"] }, "env": { "DEBUG": "1" } }

// 项目设置
{ "permissions": { "allow": ["Bash"] }, "env": { "NODE_ENV": "test" } }

// 合并结果
{
  "permissions": { "allow": ["FileEdit", "Bash"] },
  "env": { "DEBUG": "1", "NODE_ENV": "test" }
}
```

### 3.3 优先级规则总结

| 数据类型 | 合并行为 |
|----------|----------|
| **数组**（allow/deny/ask 规则等） | 跨来源连接 + 去重，无单一覆盖 |
| **对象**（env、插件配置等） | 深度合并，后来源覆盖先来源 |
| **标量**（model、defaultMode 等） | 最后来源获胜：Policy > Flag > Local > Project > User |
| **企业策略门控** | `allowManagedHooksOnly` → 仅运行托管钩子 |

---

## 四、验证机制

### 4.1 两阶段验证

```
阶段 1: 预 Schema 过滤
  → 单独验证每条权限规则
  → 移除无效规则（不影响整个文件加载）

阶段 2: Zod Schema 验证
  → safeParse() + 详细错误格式化
  → 返回 ValidationError[]
  → 包含：文件路径、字段路径（点表示法）、消息、建议、文档链接
```

### 4.2 Schema 定义模式

```typescript
// 使用 lazySchema() 处理循环依赖
export const SettingsSchema = lazySchema(() =>
  z.object({
    // ... 字段
  }).passthrough()  // 允许未知字段（保留在文件中）
)
```

### 4.3 MDM 平台支持

| 平台 | 位置 | 说明 |
|------|------|------|
| **Windows (管理员)** | `HKLM\Software\Anthropic\ClaudeCode` | 需要管理员权限 |
| **Windows (用户)** | `HKCU\Software\Anthropic\ClaudeCode` | 用户可写 |
| **macOS** | `/Library/Preferences/com.anthropic.claudecode.plist` | 管理员 |

---

## 五、Git 集成架构

### 5.1 核心文件职责

| 文件 | 职责 |
|------|------|
| `src/utils/git.ts` | 主 API（~500 行）— 仓库检测、状态读取、stash、远程 URL 规范化 |
| `src/utils/git/gitFilesystem.ts` | 基于文件系统的状态读取器（~500 行）— 零子进程开销 |
| `src/utils/git/gitConfigParser.ts` | 轻量级 .git/config 解析器 |
| `src/utils/git/gitignore.ts` | gitignore 处理与规则添加 |
| `src/utils/gitDiff.ts` | Diff 上下文构建（用于系统提示） |
| `src/utils/worktree.ts` | 工作树管理（~750 行）— Agent 隔离 |

### 5.2 核心数据结构

**GitRepoState**：
```typescript
type GitRepoState = {
  commitHash: string           // 当前 HEAD SHA
  branchName: string           // 当前分支或分离 HEAD（短 SHA）
  remoteUrl: string | null     // origin 远程 URL
  isHeadOnRemote: boolean      // @{u} 是否存在
  isClean: boolean             // 无未提交更改
  worktreeCount: number        // 活跃工作树数量
}
```

**GitDiffResult**：
```typescript
type GitDiffResult = {
  stats: {
    filesCount: number
    linesAdded: number
    linesRemoved: number
  }
  perFileStats: Map<string, {
    added: number
    removed: number
    isBinary: boolean
    isUntracked?: boolean
  }>
  hunks: Map<string, StructuredPatchHunk[]>  // 按需加载
}
```

**WorktreeSession**：
```typescript
type WorktreeSession = {
  originalCwd: string           // 工作树前的原始目录
  worktreePath: string
  worktreeName: string          // Slug 名称
  worktreeBranch?: string       // Git 分支
  originalBranch?: string       // 主仓库分支
  originalHeadCommit?: string
  sessionId: string
  tmuxSessionName?: string
  hookBased?: boolean           // 由钩子创建 vs EnterWorktree
  creationDurationMs?: number
  usedSparsePaths?: boolean     // 是否应用 sparse checkout
}
```

---

## 六、文件系统级 Git 状态读取

### 6.1 GitFileWatcher 缓存机制

```typescript
class GitFileWatcher {
  watches: ['.git/HEAD', '.git/config', 'refs/heads/<current-branch>']
  cache: Map<key, CacheEntry>  // 监控文件变化时失效

  async get<T>(key, compute) {
    if (cached && !dirty) return cached
    dirty = false
    value = await compute()
    return value
  }

  async ensureStarted() {
    if (this.initialized) return
    if (this.initPromise) return this.initPromise
    this.initPromise = this.start()  // 去重并发调用
  }
}
```

### 6.2 核心文件系统操作

```
resolveGitDir(cwd)
  → 解析 .git 路径，处理工作树（.git 文件含 gitdir: 指针）

readGitHead(gitDir)
  → 返回 {type: 'branch', name} | {type: 'detached', sha}

resolveRef(gitDir, ref)
  → 解析 git ref 到 SHA（松散文件 + packed-refs）

getCommonDir(gitDir)
  → 获取工作树的共享 .git 目录
```

### 6.3 Git 配置解析器

```typescript
// gitConfigParser.ts — 不启动子进程读取 .git/config
parseGitConfigValue(gitDir, section, subsection, key)
  → 处理引号值、转义序列、内联注释、symref 段
  → 匹配 git 源码行为（refs/files-backend.c）
```

---

## 七、Git Diff 上下文构建

### 7.1 分层策略

```
快速探测: git diff --shortstat  → O(1) 内存
详细统计: git diff --numstat    → 每文件统计
完整 Diff: git diff --          → 统一 diff 块

限制条件：
  → 最多 50 个文件
  → 每文件最大 1MB
  → 每文件最多 400 行
  → 合并/变基/cherry-pick 期间跳过
  → 空间允许时包含未跟踪文件
```

### 7.2 系统提示中的 Diff 用途

- 提供当前工作目录变更的上下文
- 帮助模型理解进行中的工作
- 支持 DiffDialog 组件的详细 hunk 渲染

---

## 八、工作树管理

### 8.1 创建流程

```
1. 验证 slug
   → validateWorktreeSlug()
   → 拒绝 . 或 .. 段、前导 /、非法字符
   → 仅允许字母数字 + 连字符 + 点 + 下划线
   → 最大 64 字符

2. 快速恢复检查
   → 读取 .git/HEAD 指针（无子进程）
   → 如果有效 SHA → 直接返回（复用）

3. 获取基础分支
   → PR: git fetch origin pull/<pr>/head
   → 常规: 先检查本地 origin/<default>
   → 回退到 git fetch
   → 环境: GIT_TERMINAL_PROMPT=0, GIT_ASKPASS=''（防挂起）

4. 创建工作树
   → git worktree add -B <branch> <path> <base>
   → -B: 从之前的删除中重置孤立分支

5. Sparse Checkout（可选）
   → git sparse-checkout set --cone <paths...>
   → git checkout HEAD
   → 在大型 monorepo 中大幅加速

6. 符号链接目录
   → node_modules, .cache, .bin（来自设置）
   → 避免复制导致的磁盘膨胀

7. 复制 .worktreeinclude 文件
   → 匹配模式的 gitignored 文件
   → 使用 git ls-files --others --ignored --directory
```

### 8.2 恢复流程（极速路径）

```
1. 检查现有 .git 指针
2. 如果有效 → 立即返回（0 次子进程调用）
3. 无 fetch、无 git 操作
4. 耗时: ~2ms（对比创建的 6-8s）
```

### 8.3 清理

```
removeWorktree(worktreeSession):
  → git worktree remove --force <path>
  → 执行删除钩子（如配置）
  → 清除会话状态
```

---

## 九、Git 安全模型

### 9.1 路径遍历防御

```typescript
// 工作树 slug 验证 — 同步执行，在任何副作用之前
function validateWorktreeSlug(slug: string): void {
  for (const segment of slug.split('/')) {
    if (segment === '.' || segment === '..') throw Error('No traversal')
    if (!VALID_WORKTREE_SLUG_SEGMENT.test(segment)) throw Error('Invalid chars')
  }
  if (slug.length > 64) throw Error('Too long')
}
```

### 9.2 Git Ref 验证

```typescript
// isSafeRefName(name) — 仅允许列表：
// ASCII 字母数字 + / . _ + - @
// 拒绝所有 Shell 元字符、空白、NUL、非 ASCII
// 防止插入 Shell 上下文时的 Git 注入

// isValidGitSha(s) — 仅接受完整 40 或 64 字符十六进制
// 防止 HEAD 或 ref 文件中的任意内容
```

### 9.3 工作树信任边界

```
规范根解析（resolveCanonicalRoot）：
1. 验证 .git 文件 → gitdir: → commondir 链
2. 确保 worktreeGitDir 是 <commonDir>/worktrees/ 的直接子项
3. 验证反向链接: <worktreeGitDir>/gitdir 指回 <gitRoot>/.git
4. realpath 解析以拒绝符号链接绕过
5. 防御恶意克隆仓库的伪工作树指针
```

---

## 十、性能特征

| 操作 | 方法 | 开销 |
|------|------|------|
| **查找 git 根** | 遍历文件系统 + LRU 缓存（50 条目） | ~10ms 首次，缓存后 <1ms |
| **获取 git 状态** | 并行 Promise.all() + 文件系统缓存 | ~15-50ms |
| **恢复工作树** | 读取 .git 指针 | ~2ms（零子进程） |
| **创建工作树** | git worktree add + fetch | 大仓库 ~6-8s，小仓库 ~1s |
| **获取 diff** | git diff --shortstat | ~100ms，>50 文件时跳过 |
| **检查 gitignore** | git check-ignore | ~50-100ms/文件 |
| **设置加载** | 多文件读取 + Zod 验证 + 缓存 | 首次 ~50-100ms，缓存后 <1ms |

---

## 十一、配置与 Git 集成点

### 11.1 设置控制 Git 行为

| 设置项 | Git 影响 |
|--------|----------|
| `includeGitInstructions` | 系统提示中包含 git 提交/PR 工作流说明 |
| `attribution.commit` | 自定义提交归属文本（空字符串隐藏） |
| `attribution.pr` | 自定义 PR 归属文本 |
| `worktree.symlinkDirectories` | 工作树创建时的符号链接目录 |
| `worktree.sparsePaths` | sparse-checkout 路径 |
| `respectGitignore` | 是否尊重 .gitignore（默认: true） |

### 11.2 权限系统与 Git

```
Git 相关权限模式：
Bash(git *)               → 所有 git 命令
Bash(git commit:*)         → 前缀匹配（旧版）
Bash(git push *)           → 通配符匹配
FileEdit(.gitignore)       → 编辑 gitignore
```

### 11.3 错误处理策略

```
优雅降级：
  → Git 子进程失败 → 返回 null，继续执行
  → 无效 git ref → 拒绝并回退
  → 工作树创建失败 → 抛出描述性错误
  → gitignore 检查失败 → 假设未忽略（关闭失败）

瞬态状态处理：
  → 合并/变基/cherry-pick/revert 期间跳过 diff
  → 这些操作暂时污染工作树
```

---

## 十二、设计原则总结

| 原则 | 配置系统实现 | Git 集成实现 |
|------|-------------|-------------|
| **层级优先** | 6 级设置优先级 + 深度合并 | 文件系统缓存 + LRU memoize |
| **安全优先** | 两阶段验证 + MDM 策略 | Ref 验证 + 路径遍历防御 |
| **性能优化** | 会话缓存 + 解析缓存 | 文件读取替代子进程 |
| **优雅降级** | 无效字段不阻止加载 | 子进程失败返回 null |
| **可扩展性** | 插件设置 + 环境变量注入 | 工作树隔离 + sparse checkout |
