# 08 MCPB bundle 与 plugin 集成

> `utils/plugins/mcpbHandler.ts` (968 行) + `mcpPluginIntegration.ts` (634 行) ≈ 1600 行——MCPB (.mcpb) 单文件 bundle 格式、user_config sensitive/non-sensitive 分流、channels assistant-mode、`${user_config.X}` 变量替换。

## 8.1 MCPB 是什么

**MCPB** = MCP Bundle，由 `@anthropic-ai/mcpb` 定义的**单文件 MCP server 分发格式**：

- 扩展名 `.mcpb` / `.dxt`（兼容 Anthropic DXT 旧格式）；
- 本质是 zip：
  - `manifest.json`（McpbManifest）—— 描述 server、user_config schema；
  - 服务器代码（JS / Python / binary）；
  - 资源文件、native deps；
- Plugin 引用 `.mcpb` 路径或 URL，Claude Code 下载 → 解压 → 生成 `McpServerConfig`。

定位：

| 格式 | 分发方式 | 适合场景 |
|------|---------|---------|
| `.mcp.json` | 文本配置 | 用户手动配置 + npm/pypi 包 |
| `.mcpb` | 单文件 bundle | Plugin marketplace 分发 + 配置驱动 |
| inline | manifest 直写 | Plugin manifest 内嵌 |

`.mcpb` 的核心价值：**可以打包 native binary**（chmod +x 保留）、**user_config schema 驱动配置 UI**。

## 8.2 isMcpbSource —— 入口判定

`mcpbHandler.ts:79`：

```ts
export function isMcpbSource(source: string): boolean {
  return source.endsWith('.mcpb') || source.endsWith('.dxt')
}
```

仅按扩展名——`.dxt` 是 Anthropic Desktop Extension（早期格式），兼容存在。

## 8.3 加载流水线

```
loadMcpbFile(source, pluginPath, pluginId)
  ├─ checkMcpbChanged() —— mtime / URL 检查
  ├─ if changed:
  │   ├─ downloadMcpb(url) or readFile(local)
  │   ├─ generateContentHash(data) —— sha256 前 16 字符
  │   ├─ parseAndValidateManifestFromBytes
  │   ├─ unzipFile → { files, modes }
  │   ├─ extractMcpbContents(files, extractPath, modes)
  │   └─ saveCacheMetadata
  ├─ existingConfig = loadMcpServerUserConfig(pluginId, serverName)
  ├─ validate(existingConfig, manifest.user_config)
  ├─ if invalid:
  │   └─ return { status: 'needs-config', configSchema, validationErrors }
  └─ generateMcpConfig(manifest, extractPath, userConfig)
      └─ return { manifest, mcpConfig, extractedPath, contentHash }
```

## 8.4 user_config sensitive/non-sensitive 分流

### 安全 incident 驱动

`mcpbHandler.ts:180-184` 注释：

> Without this split, per-channel `sensitive: true` was a false sense of security — the dialog masked the input but the save went to plaintext settings.json anyway. H1 #3617646 (Telegram/Discord bot tokens in world-readable .env) surfaced this as the gap to close.

H1 #3617646 是真实漏洞——Telegram/Discord bot token 被存到**世界可读的 .env**。dialog 显示成 password 框但**保存到明文文件**，是**虚假的安全感**。

修复：**按 schema `sensitive: true` 字段分流**：

```ts
// mcpbHandler.ts:203
for (const [key, value] of Object.entries(config)) {
  if (schema[key]?.sensitive === true) {
    sensitive[key] = String(value)
  } else {
    nonSensitive[key] = value
  }
}
```

- `sensitive: true` → secureStorage（macOS keychain / Linux `.credentials.json` 0600）
- 其它 → settings.json（明文）

### 双向 scrub 处理 schema flip

`mcpbHandler.ts:212-218`：

> Scrub ONLY keys we're writing in this call. Covers both directions across schema-version flips:
>  - sensitive→secureStorage ⇒ remove stale plaintext from settings.json
>  - nonSensitive→settings.json ⇒ remove stale entry from secureStorage
>    (otherwise loadMcpServerUserConfig's {...nonSensitive, ...sensitive} would let the stale secureStorage value win on next read)

**plugin 升级时 schema 可能改变** —— 某个字段从 `sensitive: true` 变成 false，或反过来。两边都要 scrub：

- 旧 sensitive 现 nonSensitive：从 secureStorage 删，写到 settings.json；
- 旧 nonSensitive 现 sensitive：从 settings.json 删，写到 secureStorage。

否则 `loadMcpServerUserConfig` 的 `{...nonSensitive, ...sensitive}` merge 会让**陈旧的 secureStorage 值赢**。

### Partial config 防御

> Partial `config` (user only re-enters one field) leaves other fields untouched in BOTH stores — defense-in-depth against future callers.

用户只重新输入某个字段——其它字段在两个 store 都不动。设计上让 partial save 不破坏 invariants。

### 写顺序：sensitive 先

`mcpbHandler.ts:222-225`：

> Sensitive → secureStorage FIRST. If this fails (keychain locked, .credentials.json perms), throw before touching settings.json — the old plaintext stays as a fallback instead of losing BOTH copies.

**先写 secureStorage** —— 如果 keychain 锁了 / 权限错，直接 throw。**老的明文 fallback 保留**——比同时丢两份好。

### settings.json delete-via-undefined

`mcpbHandler.ts:283-289`：

> updateSettingsForSource does mergeWith(diskSettings, ourSettings, ...) which PRESERVES destination keys absent from source — so simply omitting sensitive keys doesn't scrub them, the disk copy merges back in. Instead: set each sensitive key to explicit `undefined` — mergeWith (with the customizer at settings.ts:349) treats explicit undefined as a delete.

`updateSettingsForSource` 是 deep-merge——简单从对象里**省略 key 不够**，磁盘上的 key 会 merge 回来。**显式 `undefined` 才会 delete**——靠 mergeWith customizer 解析。

```ts
const scrubbed = Object.fromEntries(
  keysToScrubFromSettings.map(k => [k, undefined]),
) as Record<string, undefined>
settings.pluginConfigs[pluginId].mcpServers![serverName] = {
  ...nonSensitive,
  ...scrubbed,  // 这些 key 显式 undefined 触发删除
} as UserConfigValues
```

类型 cast 是必要的——`UserConfigValues` 类型不允许 undefined，但 mergeWith 需要它。注释引用 `pluginOptionsStorage.ts:184` 的同样模式。

## 8.5 serverSecretsKey 命名

`mcpbHandler.ts:124`：

```ts
function serverSecretsKey(pluginId: string, serverName: string): string {
  return `${pluginId}/${serverName}`
}
```

注释（mcpbHandler.ts:115-122）：

> Compose the secureStorage key for a per-server secret bucket. `pluginSecrets` is a flat map — per-server secrets share it with top-level plugin options (pluginOptionsStorage.ts) using a `${pluginId}/${server}` composite key. `/` can't appear in plugin IDs (`name@marketplace`) or server names (MCP identifier constraints), so it's unambiguous. Keeps the SecureStorageData schema unchanged and the single-keychain-entry size budget (~2KB stdin-safe, see INC-3028) shared across all plugin secrets.

4 个设计判断：

### 1. 用 `/` 分隔器

`pluginId` 格式 `name@marketplace`，server name 受 MCP identifier 约束（不能含 `/`）—— **`/` 在两边都不合法**，做分隔器无歧义。

### 2. 复用 `pluginSecrets` flat map

不新增 schema 字段——和 top-level plugin options 共享 store。复合 key 区分。

### 3. INC-3028 2KB stdin 限制

注释引用的 incident——单个 keychain 条目 stdin 大小有 ~2KB 限制（macOS 命令行特性）。所有 plugin secrets 共享这个预算——大量 sensitive config 可能超限。

### 4. schema 不变

`SecureStorageData` 不需要新增 `mcpServerSecrets` 字段——保持 backward-compatible。

## 8.6 文件解压：chmod +x 保留

`mcpbHandler.ts:550-617 extractMcpbContents`：

```ts
const mode = modes[filePath]
if (mode && mode & 0o111) {
  // Swallow EPERM/ENOTSUP (NFS root_squash, some FUSE mounts) — losing +x
  // is the pre-PR behavior and better than aborting mid-extraction.
  await chmod(fullPath, mode & 0o777).catch(() => {})
}
```

`modes` 来自 `parseZipModes` —— MCPB zip 保留原始 unix mode。如果有 exec bit (`0o111`)，**chmod 加回 +x**。

`.catch(() => {})` 吞下 EPERM/ENOTSUP—— NFS root_squash 或 FUSE 挂载可能拒绝 chmod，但**比中断解压好**。退化为"没有 exec bit"——pre-PR 行为。

### 过滤目录条目

```ts
const entries = Object.entries(unzipped).filter(([k]) => !k.endsWith('/'))
```

> Directory entries (common in zip -r, Python zipfile, Java ZipOutputStream) are filtered above — writeFile would create `bin/` as an empty regular file, then mkdir for `bin/server` would fail with ENOTDIR.

zip 格式允许显式目录条目（`bin/`），但 writeFile 会把它写成空文件——后续 `mkdir(bin/server)` 会 ENOTDIR。

### text vs binary 判断

```ts
const isTextFile = filePath.endsWith('.json') || ... || .endsWith('.yaml')
if (isTextFile) {
  await writeFile(fullPath, new TextDecoder().decode(fileData), 'utf-8')
} else {
  await writeFile(fullPath, Buffer.from(fileData))
}
```

text 文件 utf-8 解码——但 native binary 用 Buffer 直写避免 corruption。

按扩展名判定——**够用**：MCPB bundle 里的 binary 通常没扩展名或 `.so/.dylib/.exe`。

## 8.7 mtime 检查 + 浮点精度

`mcpbHandler.ts:670-674`：

```ts
const cachedTime = new Date(metadata.cachedAt).getTime()
// Floor to match the ms precision of cachedAt (ISO string). Sub-ms
// precision on mtimeMs would make a freshly-cached file appear "newer"
// than its own cache timestamp when both happen in the same millisecond.
const fileTime = Math.floor(stats.mtimeMs)

if (fileTime > cachedTime) { /* re-extract */ }
```

**`mtimeMs` 有亚毫秒精度** ——`new Date().getTime()` 只到 ms。如果文件 cache 和 mtime 在同 ms：

- `cachedTime = 1700000000000`
- `mtimeMs = 1700000000000.234`
- 不 `Math.floor` 就 `0.234 > 0` → 误判修改 → 反复重新解压

`Math.floor` 让两侧精度对齐——避免**"刚 cache 完又显得 modified"**死循环。

## 8.8 telemetry 发射时机

`mcpbHandler.ts:510-514`：

```ts
const data = new Uint8Array(response.data)
// Fire telemetry before writeFile — the event measures the network
// fetch, not disk I/O. A writeFile EACCES would otherwise match
// classifyFetchError's /permission denied/ → misreport as auth.
logPluginFetch('mcpb', url, 'success', performance.now() - started)
fetchTelemetryFired = true

await writeFile(destPath, Buffer.from(data))
```

**telemetry 发在 writeFile 前**——避免把磁盘 EACCES 误归类成 fetch 的"permission denied"（auth 错误）。

`fetchTelemetryFired` flag 让 catch 块知道 success 是否已上报——避免 success 后 writeFile 失败时**双发 telemetry**。

## 8.9 needs-config 状态

`mcpbHandler.ts:50-58`：

```ts
export type McpbNeedsConfigResult = {
  status: 'needs-config'
  manifest: McpbManifest
  extractedPath: string
  contentHash: string
  configSchema: UserConfigSchema
  existingConfig: UserConfigValues
  validationErrors: string[]
}
```

如果 user_config schema 必填字段未填——返回 `needs-config` 而**不是 throw**。

`mcpPluginIntegration.ts:55-64`：

```ts
if ('status' in result && result.status === 'needs-config') {
  logForDebugging(`MCPB ${mcpbPath} requires user configuration. ` +
    `User can configure via: /plugin → Manage plugins → ${plugin.name} → Configure`)
  return null  // 跳过这个 server，不是 error
}
```

UI 通过 `/plugin → Configure` 弹出 dialog 让用户填——填完重试 loadMcpbFile。

这种**非阻塞失败**让 Claude Code 能启动其它正确配置的 MCP server，**而不是因为一个未配置 plugin 卡住所有**。

## 8.10 channels 多 server 内嵌

### channels vs mcpServers

Plugin manifest 有两种声明 MCP server：

```jsonc
// plugin manifest
{
  "name": "my-plugin",
  "mcpServers": {  // 单 server
    "main": { "type": "stdio", "command": "node", ... }
  },
  "channels": [   // 多 server + per-channel userConfig
    {
      "server": "github",
      "displayName": "GitHub",
      "userConfig": {
        "token": { "type": "string", "sensitive": true, "required": true }
      }
    },
    {
      "server": "linear",
      "displayName": "Linear",
      "userConfig": {
        "apiKey": { "type": "string", "sensitive": true }
      }
    }
  ]
}
```

**channels 是 assistant-mode 概念**——单 plugin 提供多个 MCP server，每个独立 userConfig。

### getUnconfiguredChannels

`mcpPluginIntegration.ts:290-318`：

```ts
export function getUnconfiguredChannels(plugin: LoadedPlugin): UnconfiguredChannel[] {
  const channels = plugin.manifest.channels
  if (!channels?.length) return []

  const pluginId = plugin.repository
  const unconfigured: UnconfiguredChannel[] = []

  for (const channel of channels) {
    if (!channel.userConfig || Object.keys(channel.userConfig).length === 0) {
      continue  // 没有 schema 没东西问
    }
    const saved = loadMcpServerUserConfig(pluginId, channel.server) ?? {}
    const validation = validateUserConfig(saved, channel.userConfig)
    if (!validation.valid) {
      unconfigured.push({
        server: channel.server,
        displayName: channel.displayName ?? channel.server,
        configSchema: channel.userConfig,
      })
    }
  }
  return unconfigured
}
```

Plugin enable 后调一次——返回需要补充配置的 channels 列表。`ManagePlugins.tsx` 用这个决定是否弹 dialog。

### 优先级：channel > top-level

`mcpPluginIntegration.ts:440-457 buildMcpUserConfig`：

```ts
function buildMcpUserConfig(plugin, serverName): UserConfigValues | undefined {
  const topLevel = plugin.manifest.userConfig
    ? loadPluginOptions(getPluginStorageId(plugin))
    : undefined
  const channelSpecific = loadChannelUserConfig(plugin, serverName)

  if (!topLevel && !channelSpecific) return undefined
  return { ...topLevel, ...channelSpecific }  // channel 后写入 → win
}
```

> Channel-specific wins on collision so plugins that declare the same key at both levels get the more specific value.

### topLevel guard 避免无谓 keychain 读

```ts
const topLevel = plugin.manifest.userConfig
  ? loadPluginOptions(getPluginStorageId(plugin))
  : undefined
```

注释（mcpPluginIntegration.ts:444-450）：

> Gate on manifest.userConfig. loadPluginOptions always returns at least {} (it spreads two `?? {}` fallbacks), so without this guard topLevel is never undefined — the `!topLevel` check below is dead, we return {} for unconfigured plugins, and resolvePluginMcpEnvironment runs substituteUserConfigVariables against an empty map → throws on any ${user_config.X} ref. The manifest check also skips the unconditional keychain read (~50-100ms on macOS) for plugins that don't use options.

3 重价值：

1. **逻辑正确**：没有 manifest.userConfig 时 topLevel 必须是 undefined，否则 `${user_config.X}` 引用会 throw；
2. **性能**：避免无谓 keychain 读（macOS 50-100ms）；
3. **早返回**：`!topLevel && !channelSpecific` 返回 undefined → `resolvePluginMcpEnvironment` 跳过 substituteUserConfigVariables。

## 8.11 `${VAR}` 变量替换三层

`mcpPluginIntegration.ts:465-490 resolvePluginMcpEnvironment`：

```ts
const resolveValue = (value: string): string => {
  // 1. plugin-specific
  let resolved = substitutePluginVariables(value, plugin)
  // 2. user config
  if (userConfig) {
    resolved = substituteUserConfigVariables(resolved, userConfig)
  }
  // 3. general env vars
  const { expanded, missingVars } = expandEnvVarsInString(resolved)
  allMissingVars.push(...missingVars)
  return expanded
}
```

**3 层替换、顺序敏感**：

| 优先级 | 变量 | 例 | 来源 |
|--------|------|----|----|
| 1 | `${CLAUDE_PLUGIN_ROOT}` | plugin 安装路径 | plugin.path |
| 1 | `${CLAUDE_PLUGIN_DATA}` | plugin 数据目录 | getPluginDataDir |
| 2 | `${user_config.X}` | 用户配置字段 | loaded userConfig |
| 3 | `${OTHER_ENV}` | 普通环境变量 | process.env |

顺序：plugin > user_config > env。前面替换的结果**不会被后面的 env 再次展开**（如果 user_config 值含 `$FOO`，不会从 env 取）——避免 user_config 注入。

注释（mcpPluginIntegration.ts:484-485）：

> This is done last so plugin-specific and user config vars take precedence

### CLAUDE_PLUGIN_ROOT/DATA 注入 env

`mcpPluginIntegration.ts:511-515`：

```ts
const resolvedEnv: Record<string, string> = {
  CLAUDE_PLUGIN_ROOT: plugin.path,
  CLAUDE_PLUGIN_DATA: getPluginDataDir(plugin.source),
  ...(stdioConfig.env || {}),
}
```

stdio MCP server 启动时 env 里**总是有** `CLAUDE_PLUGIN_ROOT` / `CLAUDE_PLUGIN_DATA`——子进程可以读 plugin 内的资源 / 写数据目录。

## 8.12 addPluginScopeToServers —— 命名前缀

`mcpPluginIntegration.ts:341-360`：

```ts
export function addPluginScopeToServers(
  servers: Record<string, McpServerConfig>,
  pluginName: string,
  pluginSource: string,
): Record<string, ScopedMcpServerConfig> {
  const scopedServers: Record<string, ScopedMcpServerConfig> = {}

  for (const [name, config] of Object.entries(servers)) {
    const scopedName = `plugin:${pluginName}:${name}`
    const scoped: ScopedMcpServerConfig = {
      ...config,
      scope: 'dynamic',
      pluginSource,
    }
    scopedServers[scopedName] = scoped
  }

  return scopedServers
}
```

**命名前缀 `plugin:<pluginName>:<serverName>`** —— 避免不同 plugin 同名 server 冲突。`scope: 'dynamic'` —— [01 config](./01-McpServerConfig与8种transport-schema.md) 提过的 7 种 scope 之一，表示**程序动态注入**而非用户/项目配置。

最终在 [06 MCPTool 代理与命名前缀](./06-MCPTool代理与命名前缀.md) normalize 后变 `mcp__plugin_my_plugin_main__create_issue` 这种 tool name。

## 8.13 generateMcpConfig —— manifest → McpServerConfig

`mcpbHandler.ts:413-438`：

```ts
async function generateMcpConfig(
  manifest: McpbManifest,
  extractedPath: string,
  userConfig: UserConfigValues = {},
): Promise<McpServerConfig> {
  // Lazy import: @anthropic-ai/mcpb barrel pulls in zod v3 schemas (~700KB of
  // bound closures). See dxt/helpers.ts for details.
  const { getMcpConfigForManifest } = await import('@anthropic-ai/mcpb')
  const mcpConfig = await getMcpConfigForManifest({
    manifest,
    extensionPath: extractedPath,
    systemDirs: getSystemDirectories(),
    userConfig,
    pathSeparator: '/',
  })
  if (!mcpConfig) throw new Error(...)
  return mcpConfig as McpServerConfig
}
```

### lazy import

`@anthropic-ai/mcpb` 是**重量级 barrel**——700KB zod v3 schemas。

**只在需要时 import** —— Claude Code 启动不加载，没用 MCPB 的用户不付这个成本。`dxt/helpers.ts` 同模式。

### platform/systemDirs 注入

`getSystemDirectories()` 返回 `~/.config`、`~/Library/Application Support` 等系统路径——manifest 模板可以引用 `${SYSTEM_CONFIG_DIR}`。

`pathSeparator: '/'` 统一用 forward slash——manifest 写路径用 `/`，跨平台 mcpb 库自动转。

## 8.14 几个隐性设计判断

### 1. .mcpb / .dxt 双扩展名兼容

DXT 是早期 Anthropic Desktop Extension，MCPB 是规范化命名——同样格式，双扩展名兼容。

### 2. sensitive/non-sensitive 分流（H1 #3617646 驱动）

真实漏洞驱动设计——dialog 显示成 password 但**保存到明文**是虚假安全。schema 字段直接驱动存储分流。

### 3. 双向 scrub 处理 schema flip

plugin 升级时 sensitive 字段可能改变——两边都 scrub 让 merge invariant 不破。

### 4. 写顺序：sensitive 先

keychain 失败时**老明文留作 fallback**——比两份都丢好。

### 5. settings.json 删除靠显式 undefined

deep-merge 不会因省略 key 而删——必须 explicit undefined 触发 customizer。

### 6. serverSecretsKey 用 `/` 分隔

plugin ID + server name 都不允许 `/` —— 无歧义。复用 flat map 不动 schema。

### 7. chmod swallow EPERM

NFS / FUSE 不支持 chmod —— 退化到无 exec bit，不中断解压。

### 8. 过滤目录条目

zip 显式目录条目会 writeFile 成空文件——必须过滤。

### 9. text/binary 按扩展名判定

`.json/.js/.ts/.txt/.md/.yml/.yaml` 是 text，其它 binary。native binary 用 Buffer 直写不 corruption。

### 10. mtime Math.floor 对齐 ms 精度

`mtimeMs` 亚毫秒会让自身 cache 显示成"修改了"—— Math.floor 对齐。

### 11. telemetry 发在 writeFile 前

避免磁盘 EACCES 被分类成 fetch auth 错误。

### 12. needs-config 不是 error

返回特殊 status —— UI 弹 dialog，其它 plugin 不受影响。

### 13. channels 是 assistant-mode 多 server

单 plugin 多 server——每个独立 userConfig，dialog 分别问。

### 14. channel > top-level userConfig 优先

更具体的值赢——`{...topLevel, ...channelSpecific}`。

### 15. topLevel guard 三重价值

逻辑正确 + 性能（避免 keychain 读 50-100ms） + 早返回。

### 16. 3 层变量替换、顺序敏感

plugin > user_config > env —— 防止 user_config 注入扩展。

### 17. CLAUDE_PLUGIN_ROOT/DATA 总是注入 env

stdio MCP server 通过 env 访问 plugin 资源 / 数据目录。

### 18. plugin: 前缀防冲突

不同 plugin 同名 server 不冲突——scope: 'dynamic' 标识动态注入。

### 19. lazy import @anthropic-ai/mcpb

700KB barrel，不用 MCPB 的用户不付加载成本。

### 20. systemDirs + pathSeparator 跨平台

mcpb 库内部转换路径分隔符——manifest 作者写 `/` 就好。

## 8.15 与其它专题串联

- **[01 McpServerConfig](./01-McpServerConfig与8种transport-schema.md)**：`scope: 'dynamic'` 是 plugin 注入的 server；`expandEnvVarsInString` 服务这个变量替换流程；
- **[06 MCPTool 代理](./06-MCPTool代理与命名前缀.md)**：`plugin:<name>:<server>` → normalize → `mcp__plugin_xxx_yyy__tool`；
- **[07 OAuth 与认证](./07-OAuth与3套认证体系.md)**：MCPB bundle 配合 stdio 走子进程认证，不直接走 OAuth；
- **[swarm topic 10](../swarm/10-子代理工具调用与mcp集成.md)**：subagent 也可以引用 plugin MCP server。

## 8.16 小结

- MCPB (.mcpb / .dxt) 是单文件 MCP server bundle——manifest + 代码 + binary；
- 加载流水线：检查 → 下载/读取 → 解压 → 验证 user_config → 生成 McpServerConfig；
- **sensitive/non-sensitive 分流** —— 真实 H1 漏洞（#3617646）驱动设计，按 schema `sensitive: true` 字段分到 secureStorage / settings.json；
- **双向 scrub** 处理 plugin 升级时 schema flip —— 两边都清避免 stale 值赢；
- **写顺序：sensitive 先** —— keychain 失败时老明文留 fallback；
- **delete-via-undefined** —— settings.json deep-merge 必须 explicit undefined 才删；
- `serverSecretsKey` 用 `/` 分隔（pluginId 和 serverName 都不允许 `/`），复用 flat map 不动 schema；
- chmod swallow EPERM（NFS/FUSE） + 过滤目录条目 + text/binary 区分写——解压细节；
- `mtimeMs Math.floor` 对齐 ms 精度——避免自身 cache 显示"刚修改"死循环；
- telemetry 发在 writeFile 前——避免磁盘 EACCES 被分类成 fetch auth；
- `needs-config` 非阻塞失败——单 plugin 未配置不影响其它启动；
- `channels` 是 assistant-mode 概念——单 plugin 多 server，每个独立 userConfig；
- channel > top-level userConfig 优先，更具体赢；
- topLevel guard 三重价值（逻辑/性能/早返回）；
- 3 层变量替换顺序敏感（plugin > user_config > env）—— 防 user_config 注入；
- `CLAUDE_PLUGIN_ROOT/DATA` 总是注入 stdio env；
- `plugin:<name>:<server>` 命名前缀防冲突，scope: 'dynamic' 标识；
- lazy import @anthropic-ai/mcpb 700KB barrel，不用 MCPB 用户不付加载成本；
- systemDirs + pathSeparator 跨平台路径处理。

下一篇 → [09 channel 抽象与 elicitation](./09-channel抽象与elicitation.md)
