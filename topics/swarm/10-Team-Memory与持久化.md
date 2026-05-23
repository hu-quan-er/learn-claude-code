# 10 Team Memory 与持久化

> Swarm 不只是"多个 actor 并发"——还有跨 teammate 共享的持久状态。本篇拆 team memory 路径与 entrypoint、secret 防泄漏、teamMemorySync 后台 watcher、与 auto memory 的差别。

## 10.1 Team Memory 是什么

Team Memory 是个**项目级共享记忆**——不是 swarm 专属，但和 swarm 协同工作。

| 维度 | Auto Memory (private) | Team Memory (shared) |
|------|----------------------|---------------------|
| 位置 | `~/.claude/.../memory/` | `~/.claude/.../memory/team/` |
| Scope | 单个用户 | 团队/项目 |
| 同步 | 仅本机 | 通过 git / 其它机制（用户负责） |
| 适合存 | 个人偏好、个人项目状态 | 团队约定、共享上下文 |
| Secret 防护 | 无（个人责任） | **强制扫描**（防止上传到 git） |

回顾 [00 总览](./00-总览与代码地图.md) 的 team memory 模块：

- `memdir/teamMemPaths.ts` (292 行)：路径管理
- `memdir/teamMemPrompts.ts` (100 行)：模型 prompt 教育
- `services/teamMemorySync/`：
  - `index.ts` (1256 行) - 主同步逻辑
  - `secretScanner.ts` (324 行) - 密钥扫描
  - `teamMemSecretGuard.ts` (44 行) - FileWrite/Edit 拦截器
  - `watcher.ts` (387 行) - 文件变化监听
  - `types.ts` (156 行) - 类型定义

合计 ~2500 行——是个**相对独立的子系统**。本篇聚焦 swarm 相关的几个关键点。

## 10.2 isTeamMemoryEnabled —— 双重 gate

`teamMemPaths.ts:67-78`：

```ts
export function isTeamMemoryEnabled(): boolean {
  if (!isAutoMemoryEnabled()) {
    return false  // 1. auto memory 必须开
  }
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_herring_clock', false)
  // 2. GrowthBook gate
}
```

两个条件：

1. **auto memory 启用**：team memory 是 auto memory 的子目录，前者关了后者也用不了；
2. **`tengu_herring_clock` GrowthBook gate**：可远程关闭的实验性 feature。

注释（`teamMemPaths.ts:66-71`）：

> Team memory is a subdirectory of auto memory, so it requires auto memory to be enabled. This keeps all team-memory consumers (prompt, content injection, sync watcher, file detection) consistent when auto memory is disabled via env var or settings.

**一致性保障**——所有 team memory 消费者（prompt 教育、内容注入、同步 watcher、文件检测）共用同一 gate。避免 prompt 教育模型用 team memory 但 watcher 没启动这种不一致。

## 10.3 Team Memory 路径结构

```
~/.claude/projects/{sanitized-project-root}/memory/
├── MEMORY.md                          ← auto memory entrypoint (个人)
├── user_preferences.md                ← auto memory file (个人)
├── code_style.md                      ← auto memory file (个人)
└── team/                              ← team memory 子目录
    ├── MEMORY.md                      ← team memory entrypoint
    ├── api_conventions.md             ← team memory file
    └── deployment_notes.md            ← team memory file
```

3 个观察：

### 1. 路径是项目级 scope

`{sanitized-project-root}` 是 `cwd` 经过 sanitize 后的路径——每个项目独立 memory。

切换项目 → 不同 memory 目录。这避免不相关项目的 memory 互相污染。

### 2. team 是 auto memory 的子目录

为什么不放在独立目录？

- 项目 scope 一致（team 自动跟 auto 同一 project）；
- 文件搜索 / 索引一起处理；
- 卸载时一次性删完；
- 设计简洁。

### 3. 每个目录有独立 MEMORY.md entrypoint

`MEMORY.md` 是 memory 系统的"索引"——列出该目录下所有 memory 文件的概要。模型加载时优先看 MEMORY.md 决定要读哪些详细文件。

team memory 有自己的 MEMORY.md——和 auto memory 的 MEMORY.md 并列加载到上下文。

## 10.4 isTeamMemPath —— 路径判定

`teamMemPaths.ts:214-226`（推断）：

```ts
export function isTeamMemPath(filePath: string): boolean {
  const teamDir = getTeamMemPath()
  const resolved = path.resolve(filePath)
  return resolved.startsWith(teamDir)
}
```

简单的 prefix 判定——文件路径是否在 team memory 目录下。

注意它**不解析 symlink**——symlink 解析在另一个函数 `isRealPathWithinTeamDir` 里（`teamMemPaths.ts:183+`）做。两个判定用于不同场景：

- `isTeamMemPath`：**写入前的快速检查**（防止误把 secret 写进去）；
- `isRealPathWithinTeamDir`：**安全敏感的深度检查**（防 symlink 攻击）。

## 10.5 secret 防护链

Team memory 的核心安全考虑：**防止 secret 被写到 team memory 然后通过 git 同步给所有 collaborator**。

防护链 3 层：

```
┌─ Layer 1: Prompt 教育 ──────────────────────────────────────┐
│ teamMemPrompts.ts 教育模型:                                   │
│ "You MUST avoid saving sensitive data within shared team      │
│  memories. For example, never save API keys or user           │
│  credentials."                                                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ 模型仍然可能犯错
┌─ Layer 2: Write/Edit 拦截 ─────────────────────────────────┐
│ teamMemSecretGuard.ts: checkTeamMemSecrets                  │
│ - FileWrite/Edit 的 validateInput 调用                       │
│ - 扫描 content 是否含 secret                                 │
│ - 命中 → 拒绝写入, 返回错误                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ 模型重试时 prompt 看到错误
┌─ Layer 3: 同步前再扫描 ─────────────────────────────────────┐
│ teamMemorySync/index.ts: 同步到 git 前再次扫描               │
│ - 防止已写入的文件意外含 secret                              │
└─────────────────────────────────────────────────────────────┘
```

### Layer 1 —— Prompt 教育

`teamMemPrompts.ts` 里：

```ts
'- You MUST avoid saving sensitive data within shared team memories. For example, never save API keys or user credentials.',
```

直接告诉模型 "**MUST** avoid"——强语气措辞让模型知道这是 hard rule。

但模型仍然可能犯错——所以需要 layer 2 兜底。

### Layer 2 —— checkTeamMemSecrets

`teamMemSecretGuard.ts:14-44`：

```ts
export function checkTeamMemSecrets(
  filePath: string,
  content: string,
): string | null {
  if (feature('TEAMMEM')) {
    const { isTeamMemPath } = require('../../memdir/teamMemPaths.js')
    const { scanForSecrets } = require('./secretScanner.js')

    if (!isTeamMemPath(filePath)) return null

    const matches = scanForSecrets(content)
    if (matches.length === 0) return null

    const labels = matches.map(m => m.label).join(', ')
    return (
      `Content contains potential secrets (${labels}) and cannot be written to team memory. ` +
      'Team memory is shared with all repository collaborators. ' +
      'Remove the sensitive content and try again.'
    )
  }
  return null
}
```

3 步：

1. **路径检查**：不是 team memory 路径 → 不管；
2. **内容扫描**：调 `scanForSecrets` 看有没有匹配的 secret pattern；
3. **命中返回错误消息**——`null` 表示通过。

注释（`teamMemSecretGuard.ts:4-11`）：

> This is called from FileWriteTool and FileEditTool validateInput to prevent the model from writing secrets into team memory files, which would be synced to all repository collaborators.

**集成点**：FileWrite 和 FileEdit 工具的 `validateInput` 钩子调它。返回非 null → 工具调用拒绝。

模型看到错误消息后可以重试（去掉 secret），但不能 bypass。

### Layer 3 —— 同步时再扫描

`teamMemorySync/index.ts` (1256 行) 包含**同步到远端前的最终扫描**——防止 Layer 2 漏过的（如直接编辑文件而不经过 Claude 工具）。

## 10.6 scanForSecrets —— gitleaks 规则的 JS 移植

`secretScanner.ts:54-150+` 定义了 30+ 个 secret 规则，来自开源项目 **gitleaks**。

注释（`secretScanner.ts:44-60`）：

> Client-side secret scanner for team memory (PSR M22174).
>
> Scans content for credentials before upload so secrets never leave the user's machine. Uses a curated subset of high-confidence rules from gitleaks (https://github.com/gitleaks/gitleaks, MIT license) — only rules with distinctive prefixes that have near-zero false-positive rates are included.

**only rules with distinctive prefixes** —— 选用了**高置信度**的 secret patterns（如 `ghp_` GitHub PAT 前缀、`sk-ant-` Anthropic 前缀）。通用 keyword-context 规则（如"含 `password=`" 的）被**故意排除**——容易误报正常文本。

规则示例（按 secret 来源）：

| 类别 | 规则示例 |
|------|---------|
| **云服务** | `aws-access-token` (`AKIA...`) / `gcp-api-key` (`AIza...`) / `azure-ad-client-secret` / `digitalocean-pat` (`dop_v1_`) |
| **AI API** | `anthropic-api-key` (`sk-ant-`) / `openai-api-key` (`sk-`) / `huggingface` (`hf_`) |
| **版本控制** | `github-pat` (`ghp_`) / `github-oauth` (`gho_`) / `gitlab-pat` (`glpat-`) |
| ... | ... |

### 隐藏自己的 API key 前缀

注释里有个**有趣的反 leak 设计**：

```ts
// Anthropic API key prefix, assembled at runtime so the literal byte
// sequence isn't present in the external bundle (excluded-strings check).
// join() is not constant-folded by the minifier.
const ANT_KEY_PFX = ['sk', 'ant', 'api'].join('-')

const SECRET_RULES: SecretRule[] = [
  {
    id: 'anthropic-api-key',
    source: `\\b(${ANT_KEY_PFX}03-[a-zA-Z0-9_\\-]{93}AA)(?:[\\x60'"\\s;]|\\\\[nr]|$)`,
  },
  // ...
]
```

`sk-ant-api` 是 Anthropic API key 的标志性前缀——但**它本身**也是个敏感字符串。如果直接写在源码里，bundle 后能 grep 出来——任何"扫描 bundle 找 secret 前缀"的工具会误报。

**用 `.join('-')` 在运行时拼接**——bundle 里只有 `'sk', 'ant', 'api'` 三个独立字符串，**没有完整的 `sk-ant-api` 序列**。minifier 不会做常量折叠（运行时拼接被保留）。

这种**反 grep / 反扫描**的混淆是 secret 防护的细节——用扫描工具检测自己 secret 防护代码本身的 secret 时不该误报。

## 10.7 与 Memory 系统的对接

回顾 [overview 09 记忆系统与技能系统](../../overview/09-记忆系统与技能系统.md) 的 memory 整体——auto memory 是个完整子系统，team memory 是它的**第二个 scope**。

`teamMemPrompts.ts` 的 `buildCombinedMemoryPrompt` 函数生成同时教育**两种 scope** 的 prompt：

```ts
'## Memory scope',
'',
'There are two scope levels:',
'',
`- private: memories that are private between you and the current user. They persist across conversations with only this specific user and are stored at the root \`${autoDir}\`.`,
`- team: memories that are shared with and contributed by all of the users who work within this project directory. Team memories are synced at the beginning of every session and they are stored at \`${teamDir}\`.`,
```

模型看到两个目录路径——根据 memory 类型（user/feedback/project/reference）选择存哪个。

### 类型与 scope 推荐表（出自 prompt）

`memoryTypes.ts` 的 `TYPES_SECTION_COMBINED` 教育模型：

| 类型 | 推荐 scope |
|------|-----------|
| **user**（用户偏好） | private（每用户独立） |
| **feedback**（如何工作） | 二者皆可（看是个人偏好还是团队约定） |
| **project**（进行中的工作） | 二者皆可 |
| **reference**（外部引用） | 二者皆可 |

明确**没有 "team-only 类型"**——所有 4 个类型都能放在 team 或 private。模型按内容决定。

## 10.8 watcher —— 后台同步

`teamMemorySync/watcher.ts` (387 行) 是**文件系统 watcher**——监听 team memory 目录的变化：

```ts
// 推断
// 启动 chokidar watcher 监听 teamDir
const watcher = chokidar.watch(teamDir, { ... })

watcher.on('change', async (path) => {
  // 1. 检查这次变化是不是符合 sync 规则
  // 2. 如果是, trigger sync 到 remote
  // 3. 同步前再扫描 secret
})

watcher.on('add', async (path) => {
  // 新文件同样处理
})
```

**实时同步**——用户改了 team memory 文件 → 立即同步到远端（用户的 git repo / 外部 sync 服务）。

具体同步机制（git push / 自定义 backend）依赖配置——`teamMemorySync/index.ts` 的 1256 行实现各种同步路径。本篇不展开细节。

## 10.9 Team Memory 在 swarm 里的角色

虽然 team memory 不是 swarm 专属功能，但它**和 swarm 协同**：

### 1. 跨 teammate 共享上下文

team memory 文件在 swarm 启动时**被所有 teammate 读取**（通过标准 memory 系统）——它们看到同一份团队上下文。

例：
- `team/api_conventions.md`：API 命名约定
- `team/deployment_notes.md`：部署注意事项

researcher / coder / reviewer 三个 teammate 都看到这些——保证一致性。

### 2. teammate 也能写 team memory

teammate 调 FileWrite 写 team memory → 走 secret 防护 → 通过后写盘 → watcher 同步到远端。

如果 researcher 发现"API 用 camelCase 命名"，可以记录到 team memory：

```
researcher: "I'll save this convention to team memory"
  → FileWrite ~/.claude/.../memory/team/api_conventions.md
  → secret 防护通过
  → 文件写入
  → watcher 触发同步
  → 其它 teammate（即使在不同 session 启动的）下次都能看到
```

### 3. lead 决策时的额外上下文

lead 在弹权限对话框时——team memory 里的"团队约定"可能影响决策。例如 team memory 写了"禁止跑 rm 命令"，lead 看到 worker 请求跑 rm 时可以参考。

## 10.10 几个隐性设计判断

### 1. team memory 是 auto memory 的子目录

不是独立目录——共享 sanitize 逻辑、统一开关、一致清理。**简洁性优先**。

### 2. 双重 gate（auto memory + GrowthBook）

team memory 必须 auto memory 启用——避免不一致。GrowthBook gate 可远程关——实验性 feature 标准做法。

### 3. 3 层 secret 防护

prompt 教育 → write 拦截 → 同步前再扫——纵深防御。

**协议教育 < 工具拦截 < 同步层最终扫描**——每一层独立、不依赖前面。

### 4. 用 gitleaks 高置信度规则

只用**前缀显著**的规则（如 `ghp_` / `sk-ant-`）——避免假阳性。通用 keyword 规则被**故意排除**。

正常文档里可能含 `password=...example...`——不应该被当成 secret。

### 5. 反扫描自己的 secret prefix

`['sk', 'ant', 'api'].join('-')` 让 bundle 不含完整 `sk-ant-api` 字符串——防 grep 自己。

这种**对工具自身实现的混淆**是细节工程的标志。

### 6. 同步时再扫描

不只信任 write-time 拦截——同步前再扫一遍。**多层独立检查**——任一层修复都能挡住。

### 7. project-level scope

每个项目独立 team memory——切项目自动切 memory。避免不相关 context 污染。

### 8. 类型与 scope 解耦

4 个 memory 类型 × 2 个 scope = 8 种组合。**没有强制类型 → scope** 的映射——模型按内容决定。

### 9. watcher 实时同步

文件改了立刻同步——避免"用户改了但 teammate 没同步"的不一致。代价：sync 频率高、I/O 增加。

### 10. 路径判定的两版

`isTeamMemPath`（快速）+ `isRealPathWithinTeamDir`（深度）——不同场景用不同精度。

## 10.11 与其它专题串联

- **[overview 09 记忆系统](../../overview/09-记忆系统与技能系统.md)**：team memory 是 memory 系统的第二 scope；
- **[01 激活机制](./01-激活机制与团队配置.md)**：team config 文件位置和 team memory 文件位置在不同目录（前者 `~/.claude/teams/`、后者 `~/.claude/projects/.../memory/team/`）；
- **[bashtool 02 命令解析](../bashtool/02-命令解析管线.md)**：secret scanner 用类似的 regex 防御思路——allowlist + 高置信度模式；
- **[prompt-injection 03 Unicode 清洗](../prompt-injection/03-Unicode清洗管线.md)**：team memory 的 secret 防护是另一种 "in-bound" 防护。

## 10.12 小结

- Team Memory 是 auto memory 的第二 scope——共享给项目所有 collaborator；
- 双重 gate：auto memory 必须启用 + GrowthBook `tengu_herring_clock`；
- 路径：`~/.claude/projects/{project}/memory/team/`，与 auto memory 并列；
- 3 层 secret 防护：prompt 教育 / Write-Edit 拦截 / 同步前扫描；
- secret 规则用 gitleaks 高置信度子集——避免通用 keyword 假阳性；
- 用 `.join('-')` 隐藏自己的 secret prefix 避免 grep 自己；
- watcher 实时监听文件变化并同步；
- teammate 也能写 team memory——通过标准 FileWrite/Edit 路径走 secret 防护；
- swarm 里的角色：跨 teammate 共享上下文、决策参考、长期约定持久化。

下一篇 → [11 端到端模拟与设计哲学](./11-端到端模拟与设计哲学.md)
