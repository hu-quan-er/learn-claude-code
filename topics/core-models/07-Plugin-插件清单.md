# 07. Plugin — 插件清单与加载结果模型

> 这一篇做一件事：把 `restored-src/src/types/plugin.ts` 363 行**作为 model**讲透——`BuiltinPluginDefinition` / `LoadedPlugin` / `PluginConfig` / `PluginRepository` / `PluginError`（30+ 种 error type）/ `PluginLoadResult`，以及它们引用的 `PluginManifest`。读完之后你能解释"为什么 Plugin 系统的错误模型比成功模型更重要"。
>
> 接 06 篇：hook 是 plugin 的核心组件之一；接 04 篇：plugin 状态在 AppState.plugins 里；接 02 篇：plugin 的 tools/skills 都用 Tool / Command 抽象表达。

---

## 1. Plugin 在所有 model 里的位置

Plugin 是 Claude Code 的**第三方扩展机制**。一个 plugin 可以提供：

- Skills（`.claude/skills/<name>/SKILL.md` 同形态）
- Commands（slash commands）
- Agents（`.claude/agents/<name>.md` 同形态）
- Hooks（生命周期钩子）
- MCP servers（远程 tool）
- LSP servers（语言服务器）
- Output styles（输出风格）

来源：

- 内置 plugin（bundled with CLI）
- 用户安装的 plugin（marketplace / GitHub / local path）
- enterprise 部署的 plugin

每个 plugin 都遵循一个 **manifest schema**——schema 定义在 `utils/plugins/schemas.ts`（不在快照），但用法可见。Plugin 装载是 Claude Code **最复杂的子系统之一**——涉及 git 拉取、marketplace 查询、依赖解析、签名验证、版本固定等。

`types/plugin.ts` 是 plugin 系统**对外契约**——其它子系统通过这些类型理解 plugin 状态。

---

## 2. 文件骨架

```
types/plugin.ts (1-363)
│
├── 段 0  imports + re-exports               1-12
│
├── 段 1  BuiltinPluginDefinition            13-35      内置 plugin 定义
│
├── 段 2  PluginRepository / PluginConfig    37-46      仓库源
│
├── 段 3  LoadedPlugin                       48-70      ★ 已加载 plugin 主结构
│
├── 段 4  PluginComponent                    72-77      组件类型枚举
│
├── 段 5  PluginError                        79-283     ★ 30+ 种错误 union
│        IMPLEMENTATION STATUS 注释:
│        当前生产用 2 种，10+ 种规划中
│
├── 段 6  PluginLoadResult                   285-289    加载汇总
│
└── 段 7  getPluginErrorMessage              295-363    每种错误的 display 字符串
```

`PluginManifest` 是从 `utils/plugins/schemas.ts` re-export（行 11），定义不在快照里——但从用法能反推大致字段。

---

## 3. 段 1：BuiltinPluginDefinition（13-35）— 内置 plugin

```ts
export type BuiltinPluginDefinition = {
  name: string                                  // {name}@builtin 格式
  description: string                           // /plugin UI 显示
  version?: string
  skills?: BundledSkillDefinition[]             // 见 01 篇 bundled skill
  hooks?: HooksSettings                          // 见 06 篇
  mcpServers?: Record<string, McpServerConfig>
  isAvailable?: () => boolean                    // 系统能力检查
  defaultEnabled?: boolean                       // 默认是否启用
}
```

### 3.1 这是"内置 + 可启停"的折中

注释 14-17：

```
Built-in plugins appear in the /plugin UI and can be enabled/disabled by
users (persisted to user settings).
```

**bundled skill 永远内置**（用户不能关），**built-in plugin 是可启停的内置组件**——bundled skill 和 built-in plugin 的重要区别。

### 3.2 字段对照其它 model

- `skills: BundledSkillDefinition[]` —— 复用 bundled skill 的同 model
- `hooks: HooksSettings` —— 复用 settings hooks schema
- `mcpServers: Record<string, McpServerConfig>` —— 复用 MCP 配置

**BuiltinPluginDefinition 不发明新东西**——只是把已有 model 组合起来。这种"插件 = 已知组件的容器"是清晰的设计。

### 3.3 `isAvailable: () => boolean`

注释 31：

```
Whether this plugin is available (e.g. based on system capabilities).
Unavailable plugins are hidden entirely.
```

`Setup` plugin 可能要求 macOS / Windows / Linux；**不可用的 plugin 完全隐藏**——不是显示为 disabled，而是 UI 里根本看不到。这是用户体验细节——别让用户看到"我能开但其实开不了"的灰按钮。

---

## 4. 段 2：PluginRepository / PluginConfig（37-46）

```ts
export type PluginRepository = {
  url: string
  branch: string
  lastUpdated?: string
  commitSha?: string
}

export type PluginConfig = {
  repositories: Record<string, PluginRepository>
}
```

### 4.1 PluginRepository — git 仓库源

每个 plugin 来自一个 git 仓库（即使是 marketplace plugin 实际背后也是 git）。

字段：

- `url` — git URL（https / ssh）
- `branch` — 分支
- `lastUpdated?` — 上次更新时间
- `commitSha?` — 锁定 commit（version pin）

`commitSha` 让 plugin **可以版本锁**——用户不希望 plugin 自动更新破坏现有 workflow。

### 4.2 PluginConfig — 仓库注册表

```ts
{ repositories: Record<string, PluginRepository> }
```

按 plugin name 索引到具体 repository。这通常持久在 settings.json 里。

---

## 5. 段 3：LoadedPlugin（48-70）— 加载完成的运行时结构

```ts
export type LoadedPlugin = {
  name: string
  manifest: PluginManifest                       // 解析后的 manifest
  path: string                                   // 磁盘路径
  source: string                                 // 来源标识
  repository: string                             // 仓库 ID
  enabled?: boolean
  isBuiltin?: boolean
  sha?: string                                   // git commit SHA

  // 7 种组件路径（每种可能多个）
  commandsPath?: string
  commandsPaths?: string[]
  commandsMetadata?: Record<string, CommandMetadata>
  agentsPath?: string
  agentsPaths?: string[]
  skillsPath?: string
  skillsPaths?: string[]
  outputStylesPath?: string
  outputStylesPaths?: string[]

  // 直接配置
  hooksConfig?: HooksSettings
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>
  settings?: Record<string, unknown>
}
```

### 5.1 单数 vs 复数路径

观察 `commandsPath?: string` 和 `commandsPaths?: string[]` 共存——agents / skills / outputStyles 同样模式。

设计的逻辑：

- **单数 `xxxPath`** — plugin 默认目录（自动检测，比如 `commands/` 子目录）
- **复数 `xxxPaths`** — manifest 显式声明的额外路径

例子：plugin 根目录有 `commands/`（自动），manifest 又 declare `"commands": ["lib/extra-commands"]`——这两个路径都装进 commands。

为什么不合并成单字段 `commandsPaths: string[]`？因为单数代表"约定路径"，复数代表"显式配置"——语义上 distinct。

### 5.2 CommandMetadata — manifest 里 named command 的元数据

```ts
commandsMetadata?: Record<string, CommandMetadata>
```

`CommandMetadata` 类型来自 `utils/plugins/schemas.ts`——大致包含 description / argumentHint / hidden 等。让 plugin 作者**通过 manifest 而不是 markdown frontmatter** 给命令打元数据。

### 5.3 三种"来源 / 仓库 / 路径"字段

| 字段 | 含义 |
|---|---|
| `path` | plugin 在磁盘上的根目录 |
| `source` | 来源字符串（`"plugin@marketplace"` 类） |
| `repository` | 仓库 ID（通常等于 source） |

#### 为什么要分？

`source` 和 `repository` 同语义但**字段保留**——可能历史演化 / 未来分歧。注释 53：

```
Repository identifier, usually same as source
```

"usually same"暗示了**未来可能不同**的可能性。

### 5.4 `settings: Record<string, unknown>`

plugin 自己的配置——key 是用户在 settings 里给的（`{user_config.<plugin>.<key>}`），value 任意。

`createPluginCommand` 在 prompt 展开时用它替换 `${user_config.xxx}`（参考 `topics/skill/03-装载层逐行精读.md` 的 Command 构造部分）。

---

## 6. 段 4：PluginComponent（72-77）

```ts
export type PluginComponent =
  | 'commands'
  | 'agents'
  | 'skills'
  | 'hooks'
  | 'output-styles'
```

5 种组件类型枚举。用在 `PluginError` 里指明"哪类组件加载失败"。

注意没有 `'mcp'` / `'lsp'`——MCP/LSP 错误有自己的细分 error type（看下面）。

---

## 7. 段 5：PluginError（79-283）— 30+ 种错误 union

这是 `types/plugin.ts` 整个文件**最大的一段**——200 行。

### 7.1 IMPLEMENTATION STATUS 注释（80-100）

```
Discriminated union of plugin error types.
Each error type has specific contextual data for better debugging and user guidance.

This replaces the previous string-based error matching approach with type-safe
error handling that can't break when error messages change.

IMPLEMENTATION STATUS:
Currently used in production (2 types):
- generic-error: Used for various plugin loading failures
- plugin-not-found: Used when plugin not found in marketplace

Planned for future use (10 types - see TODOs in pluginLoader.ts):
- path-not-found, git-auth-failed, git-timeout, network-error
- manifest-parse-error, manifest-validation-error
- marketplace-not-found, marketplace-load-failed
- mcp-config-invalid, hook-load-failed, component-load-failed

These unused types support UI formatting and provide a clear roadmap for
improving error specificity. They can be incrementally implemented as
error creation sites are refactored.
```

#### 这一段揭示了一个有意思的设计哲学

**"先定义类型，后实现使用"** —— 30+ 种 error type 大部分**还没真正在生产代码里产生**。但类型已经定义好。

为什么这么做？

1. **设计文档在类型里**：每个 error type 表达"系统将来该如何区分这种失败"——比文档好维护
2. **UI 提前适配**：UI 渲染（`getPluginErrorMessage`）按 type narrow，未来 plugin loader 升级产生新 type，UI 不变就能展示
3. **重构路标**：`pluginLoader.ts` TODO 注释里指明"这里应该产生 git-auth-failed 而不是 generic-error"——逐步迁移

这种 **type-first**（类型先行）方法比 **string-error 匹配**优雅多了——后者每次错误信息改一字 UI 就要适配。

### 7.2 30+ 种 error 分类

按主题分：

#### 路径 / 网络（4）

| `type` | 字段 |
|---|---|
| `path-not-found` | source, plugin?, path, component |
| `git-auth-failed` | source, plugin?, gitUrl, authType: 'ssh' \| 'https' |
| `git-timeout` | source, plugin?, gitUrl, operation: 'clone' \| 'pull' |
| `network-error` | source, plugin?, url, details? |

#### Manifest 解析（2）

| `type` | 字段 |
|---|---|
| `manifest-parse-error` | source, plugin?, manifestPath, parseError |
| `manifest-validation-error` | source, plugin?, manifestPath, validationErrors[] |

#### Marketplace（3）

| `type` | 字段 |
|---|---|
| `plugin-not-found` | source, pluginId, marketplace |
| `marketplace-not-found` | source, marketplace, availableMarketplaces[] |
| `marketplace-load-failed` | source, marketplace, reason |

#### MCP（2）

| `type` | 字段 |
|---|---|
| `mcp-config-invalid` | source, plugin, serverName, validationError |
| `mcp-server-suppressed-duplicate` | source, plugin, serverName, duplicateOf |

#### LSP（5）

| `type` | 字段 |
|---|---|
| `lsp-config-invalid` | source, plugin, serverName, validationError |
| `lsp-server-start-failed` | source, plugin, serverName, reason |
| `lsp-server-crashed` | source, plugin, serverName, exitCode \| signal |
| `lsp-request-timeout` | source, plugin, serverName, method, timeoutMs |
| `lsp-request-failed` | source, plugin, serverName, method, error |

#### Hook / Component 加载（2）

| `type` | 字段 |
|---|---|
| `hook-load-failed` | source, plugin, hookPath, reason |
| `component-load-failed` | source, plugin, component, path, reason |

#### MCPB（MCP Bundle）（3）

| `type` | 字段 |
|---|---|
| `mcpb-download-failed` | source, plugin, url, reason |
| `mcpb-extract-failed` | source, plugin, mcpbPath, reason |
| `mcpb-invalid-manifest` | source, plugin, mcpbPath, validationError |

#### 策略 / 依赖 / 缓存（3）

| `type` | 字段 |
|---|---|
| `marketplace-blocked-by-policy` | marketplace, blockedByBlocklist?, allowedSources[] |
| `dependency-unsatisfied` | plugin, dependency, reason: 'not-enabled' \| 'not-found' |
| `plugin-cache-miss` | plugin, installPath |

#### 兜底（1）

| `type` | 字段 |
|---|---|
| `generic-error` | source, plugin?, error |

### 7.3 每种 error 的字段都精心选择

观察 LSP 错误 5 种：

- `lsp-config-invalid` — 配置阶段（serverName, validationError）
- `lsp-server-start-failed` — 启动阶段（serverName, reason）
- `lsp-server-crashed` — 运行阶段（exitCode \| signal）
- `lsp-request-timeout` — 请求阶段（method, timeoutMs）
- `lsp-request-failed` — 请求阶段（method, error）

**每种状态自带需要展示的具体字段**——`exitCode` / `signal` 给 server crashed，`timeoutMs` 给 timeout。UI 渲染时对症下药，错误消息**自然信息丰富**。

如果用 `generic-error` 装所有，"server crashed" 信息就是个字符串——UI 没法精准展示"signal SIGKILL"还是"exit code 1"。

### 7.4 `marketplace-blocked-by-policy` 的 `blockedByBlocklist` 字段

```ts
{
  type: 'marketplace-blocked-by-policy'
  source: string
  plugin?: string
  marketplace: string
  blockedByBlocklist?: boolean
  allowedSources: string[]
}
```

注释 263：

```
true if blocked by blockedMarketplaces, false if not in strictKnownMarketplaces
```

**两种"被策略拦"的语义**：

- 显式拒绝（黑名单）— `blockedByBlocklist: true`
- 隐式拒绝（不在白名单）— `blockedByBlocklist: false`

UI 渲染时区别对待——前者是"这个 marketplace 被你的 IT 部门明确拒绝"，后者是"你的 IT 部门只允许特定 marketplace"。文字差很大。

### 7.5 `dependency-unsatisfied`（266-271）

```ts
{
  type: 'dependency-unsatisfied'
  source: string
  plugin: string
  dependency: string
  reason: 'not-enabled' | 'not-found'
}
```

**plugin 之间的依赖**——某个 plugin 依赖另一个。两种失败：

- `not-enabled` — 依赖的 plugin 装了但 disabled（用户解决：启用它）
- `not-found` — 完全没装（用户解决：装它）

`getPluginErrorMessage` 的对应 case：

```ts
case 'dependency-unsatisfied': {
  const hint =
    error.reason === 'not-enabled'
      ? 'disabled — enable it or remove the dependency'
      : 'not found in any configured marketplace'
  return `Dependency "${error.dependency}" is ${hint}`
}
```

**针对每种 reason 给具体修复建议**——好的错误消息不只说"失败了"，还说"接下来该做什么"。

---

## 8. 段 6：PluginLoadResult（285-289）

```ts
export type PluginLoadResult = {
  enabled: LoadedPlugin[]
  disabled: LoadedPlugin[]
  errors: PluginError[]
}
```

#### 三态分类

- `enabled` — 加载成功且用户启用
- `disabled` — 加载成功但用户关闭
- `errors` — 加载失败

为什么 disabled 也保留 LoadedPlugin？因为 `/plugin` UI 要展示"我有哪些 plugin（含 disabled），点开能看到详情"。**完全丢掉 disabled** 让 UI 没法再次启用。

`errors` 是 PluginError[] 列表——多个 plugin 同时出问题全部记录。

---

## 9. 段 7：getPluginErrorMessage（295-363）

```ts
export function getPluginErrorMessage(error: PluginError): string {
  switch (error.type) {
    case 'generic-error':
      return error.error
    case 'path-not-found':
      return `Path not found: ${error.path} (${error.component})`
    case 'git-auth-failed':
      return `Git authentication failed (${error.authType}): ${error.gitUrl}`
    // ... 30+ 个 case
  }
}
```

### 9.1 一个 helper 把 30+ 种 error 转 string

每个 case 用**该 type 暴露的字段**构造消息：

```ts
case 'lsp-server-crashed':
  if (error.signal) {
    return `Plugin "${error.plugin}" LSP server "${error.serverName}" crashed with signal ${error.signal}`
  }
  return `Plugin "${error.plugin}" LSP server "${error.serverName}" crashed with exit code ${error.exitCode ?? 'unknown'}`
```

注意 TypeScript 在 switch case 里**自动 narrow**——`error.signal` 在 `'lsp-server-crashed'` case 里能用，因为 union 已经判别。

### 9.2 `mcp-server-suppressed-duplicate` 的特殊处理

```ts
case 'mcp-server-suppressed-duplicate': {
  const dup = error.duplicateOf.startsWith('plugin:')
    ? `server provided by plugin "${error.duplicateOf.split(':')[1] ?? '?'}"`
    : `already-configured "${error.duplicateOf}"`
  return `MCP server "${error.serverName}" skipped — same command/URL as ${dup}`
}
```

`duplicateOf` 是字符串——可能是 `"plugin:my-plugin"` 也可能是用户自配置的服务器名。这里**按前缀分流处理**——产出两种不同表达。

### 9.3 这种"errors-as-types-with-formatters"模式的好处

1. **类型安全**：`switch` 必须穷尽所有 case，否则 TS 报错
2. **字段不丢**：每个 case 都用到 type 的所有上下文字段
3. **国际化友好**：把 `getPluginErrorMessage` 替换成 `getPluginErrorMessage_zhCN` 就能切语言（虽然当前没看到 i18n）
4. **测试友好**：每个 case 是纯函数——unit test 直接断言输出字符串

---

## 10. PluginManifest（提到但定义在快照外）

`types/plugin.ts:11` re-export：

```ts
export type { PluginAuthor, PluginManifest, CommandMetadata }
```

这三个类型来自 `utils/plugins/schemas.ts`（不在快照）。从用法反推：

```
PluginManifest = {
  name: string
  version?: string
  description?: string
  author?: PluginAuthor

  // 组件声明
  commands?: string | string[]            // 路径
  agents?: string | string[]
  skills?: string | string[]
  outputStyles?: string | string[]

  hooks?: HooksSettings | string          // 内联或路径
  mcpServers?: Record<string, McpServerConfig>
  lspServers?: Record<string, LspServerConfig>

  dependencies?: Record<string, string>   // plugin 间依赖

  // 用户配置 schema
  configSchema?: ...                      // JSON schema
}

PluginAuthor = { name?: string; email?: string; url?: string }

CommandMetadata = {
  description?: string
  argumentHint?: string
  hidden?: boolean
  // ...
}
```

具体字段以实际 schema 为准，但形态大致如此。

---

## 11. Plugin 在 model 网里的关系

```
                        用户配置
                  ┌─────────────────┐
                  │ settings.json   │
                  │ "enabledPlugins"│
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────────────┐
                  │ PluginConfig             │
                  │   repositories           │
                  └────────┬────────────────┘
                           │ git clone / pull
                           ▼
                  ┌─────────────────────────┐
                  │ 磁盘上的 plugin 目录     │
                  │  - manifest.json         │
                  │  - commands/             │
                  │  - skills/               │
                  │  - agents/               │
                  │  - hooks                 │
                  └────────┬────────────────┘
                           │ load
                           ▼
                  ┌─────────────────────────┐
                  │ PluginLoadResult         │
                  │   enabled: LoadedPlugin[]│
                  │   disabled: LoadedPlugin[]│
                  │   errors: PluginError[]  │ ← 30+ 种
                  └────────┬────────────────┘
                           │
                           ▼
                  ┌─────────────────────────┐
                  │ AppState.plugins         │
                  │  (04 篇讲过)              │
                  └────┬───┬───────┬────────┘
                       │   │       │
            ┌──────────┘   │       └─────────┐
            │              │                  │
            ▼              ▼                  ▼
        commands         skills              hooks
        Command[]       Command[]          HooksSettings
        ↓ 01 篇          ↓ skills 篇        ↓ 06 篇
            │              │                  │
            └──────────────┴──────────────────┘
                           │
                           ▼
                  Tool / Command 系统
                  执行调用、权限检查、UI 渲染
```

---

## 12. 这一篇你应该带走的几样东西

1. BuiltinPluginDefinition vs LoadedPlugin —— 内置定义 vs 已加载状态
2. LoadedPlugin 的"单数路径 vs 复数路径"模式——约定 vs 显式配置
3. PluginRepository 的 `commitSha` 让 plugin 可以版本锁
4. PluginError 30+ 种 type 的 IMPLEMENTATION STATUS——type-first 设计哲学
5. 每种 error 的字段精心选择（exitCode \| signal、timeoutMs 等）
6. `marketplace-blocked-by-policy.blockedByBlocklist` 区分黑名单 vs 白名单语义
7. `dependency-unsatisfied.reason` 区分 not-enabled vs not-found，对应不同修复建议
8. PluginLoadResult 三态（enabled / disabled / errors）让 UI 全功能
9. `getPluginErrorMessage` 用 switch + TS narrow 把 30+ error 转 string，类型安全
10. 5 种 PluginComponent 不含 mcp/lsp——这两个有自己的 error 细分

---

## 13. 设计哲学要点

1. **type-first 错误设计**：先定义类型，再逐步实现产出位点。类型本身就是设计文档。
2. **错误字段精确建模**：不用 `error: string` 兜底，按错误状态选最有信息量的字段。
3. **switch 强制穷尽**：discriminated union 让新加 error type 必须更新 formatter，编译期发现。
4. **三态加载结果**：enabled / disabled / errors 分类——保留全部用户可见信息。
5. **路径双轨**：约定路径（单数）+ 显式路径（复数）共存，语义不同。
6. **版本锁**：commitSha 字段让用户能选择"不要自动更新"。
7. **不发明轮子**：BuiltinPluginDefinition 的 skills/hooks/mcpServers 复用其它 model，不发明新 schema。
8. **UI-friendly 错误消息**：每个 error 自带"接下来怎么办"的提示。

下一篇 08 看**设计哲学综合提炼**——把前 7 篇看到的所有 model 设计选择串起来，看 Claude Code 是怎么思考问题的。
