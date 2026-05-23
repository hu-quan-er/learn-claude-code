# 07 Sandbox 沙箱机制

> 即使前面 4 道闸门都漏过，sandbox 仍能限制命令真正能做什么。本篇拆 `shouldUseSandbox.ts` 153 行的判定路径、`sandbox-adapter.ts` 985 行的 ISandboxManager 接口、Darwin sandbox-exec / Linux 平台差异。

## 7.1 沙箱在防御链路里的位置

回顾 [00 总览](./00-总览与代码地图.md) 的 5 道闸门——闸门 4 就是沙箱。它的特点：

- **不在乎命令名**：跑 `cat` 还是 `rm`，沙箱只看你访问的文件路径、网络端口；
- **行为面收紧**：限制进程能 read / write 哪些路径、能连哪些主机；
- **OS 级隔离**：用 Darwin sandbox-exec / Linux 各种机制——攻击者无法用 shell 技巧绕过；
- **最后一道防线**：即使前面所有闸门误放行，沙箱仍限制能造成的破坏面。

## 7.2 shouldUseSandbox —— 决策路径

`shouldUseSandbox.ts:130-153` 是入口：

```ts
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  if (!SandboxManager.isSandboxingEnabled()) {
    return false
  }

  // Don't sandbox if explicitly overridden AND unsandboxed commands are allowed by policy
  if (
    input.dangerouslyDisableSandbox &&
    SandboxManager.areUnsandboxedCommandsAllowed()
  ) {
    return false
  }

  if (!input.command) {
    return false
  }

  // Don't sandbox if the command contains user-configured excluded commands
  if (containsExcludedCommand(input.command)) {
    return false
  }

  return true
}
```

4 步判定：

1. **整体启用？** 用户在设置里关了沙箱 → 直接 false；
2. **dangerouslyDisableSandbox + 政策允许？** 模型在 input 里设了 `dangerouslyDisableSandbox: true`，且**政策**允许跳过 → false；
3. **没有 command？** → false（什么都不跑无所谓沙箱）；
4. **excluded commands 命中？** → false（用户配置的"已知信任"命令）；

否则 → **true（用沙箱）**。

注意 step 2 的 **两个条件叠加**：

```ts
input.dangerouslyDisableSandbox && SandboxManager.areUnsandboxedCommandsAllowed()
```

模型自己设 `dangerouslyDisableSandbox: true` 不够——还要 `areUnsandboxedCommandsAllowed()`（用户/管理员级政策）也允许。**双重许可**避免模型单方面跳出沙箱。

## 7.3 containsExcludedCommand —— 多源 excluded 列表

`shouldUseSandbox.ts:21-128` 是 excluded command 的判定。3 个来源：

### 来源 1：dynamic config（ant-only）

```ts
if (process.env.USER_TYPE === 'ant') {
  const disabledCommands = getFeatureValue_CACHED_MAY_BE_STALE<{
    commands: string[]
    substrings: string[]
  }>('tengu_sandbox_disabled_commands', { commands: [], substrings: [] })
  // ...
}
```

ant 内部通过 `tengu_sandbox_disabled_commands` GrowthBook feature 动态配置——出问题随时禁用沙箱包装某个命令。外部 build 完全不走这条路径（DCE 消除）。

### 来源 2：user settings `sandbox.excludedCommands`

```ts
const settings = getSettings_DEPRECATED()
const userExcludedCommands = settings.sandbox?.excludedCommands ?? []
```

用户在 `~/.claude/settings.json` 里配置的 `sandbox.excludedCommands` 数组。一旦命令匹配 → 不进沙箱。

### 复合命令逐 subcommand 检查

```ts
// Split compound commands (e.g. "docker ps && curl evil.com") into individual
// subcommands and check each one against excluded patterns. This prevents a
// compound command from escaping the sandbox just because its first subcommand
// matches an excluded pattern.
let subcommands: string[]
try {
  subcommands = splitCommand_DEPRECATED(command)
} catch {
  subcommands = [command]
}

for (const subcommand of subcommands) {
  const trimmed = subcommand.trim()
  // ... matching logic
}
```

**任一 subcommand 不匹配 excluded → 整体仍走沙箱**。换句话说：

```bash
docker ps && curl evil.com
```

如果 `excludedCommands` 配了 `docker ps`，整体仍**进沙箱**——因为 `curl evil.com` 不在 excluded 列表。这避免攻击者用复合命令"挂载"一个 excluded 命令来逃出沙箱。

### Fixed-point iteration —— 同 04 篇

```ts
const candidates = [trimmed]
const seen = new Set(candidates)
let startIdx = 0
while (startIdx < candidates.length) {
  const endIdx = candidates.length
  for (let i = startIdx; i < endIdx; i++) {
    const cmd = candidates[i]!
    const envStripped = stripAllLeadingEnvVars(cmd, BINARY_HIJACK_VARS)
    if (!seen.has(envStripped)) {
      candidates.push(envStripped)
      seen.add(envStripped)
    }
    const wrapperStripped = stripSafeWrappers(cmd)
    if (!seen.has(wrapperStripped)) {
      candidates.push(wrapperStripped)
      seen.add(wrapperStripped)
    }
  }
  startIdx = endIdx
}
```

和 [04 permission 决策](./04-permission决策核心.md#4-fixed-point-迭代处理多层-wrapper--env-var-包装) 一样的迭代策略——直到没有新候选产生。这处理 `nohup FOO=bar timeout 5 bazel` 这类多层 wrapper + env var 嵌套。

注意 `stripAllLeadingEnvVars` 用了 `BINARY_HIJACK_VARS` 作为 blocklist——遇到 `LD_*` / `DYLD_*` / `PATH=` 这类劫持变量**不剥**，让规则按"带前缀"形式失败（保险）。

### 重要 disclaimer

注释（`shouldUseSandbox.ts:18-20`）：

> NOTE: excludedCommands is a user-facing convenience feature, not a security boundary. It is not a security bug to be able to bypass excludedCommands — the sandbox permission system (which prompts users) is the actual security control.

**excludedCommands 不是安全边界**——它是"用户已知信任的命令跳过沙箱"的便利特性。真正的安全控制是 permission 系统（弹窗给用户）。

这种**明确划清"安全边界 vs 便利特性"** 的注释是工程成熟度的标志——避免未来工程师误把 convenience 当 security 加固。

## 7.4 ISandboxManager 接口 —— 30+ 方法

`sandbox-adapter.ts:880-922` 定义了沙箱管理器的完整接口：

```ts
export interface ISandboxManager {
  initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void>

  // 平台与启用判定
  isSupportedPlatform(): boolean
  isPlatformInEnabledList(): boolean
  getSandboxUnavailableReason(): string | undefined
  isSandboxingEnabled(): boolean
  isSandboxEnabledInSettings(): boolean
  checkDependencies(): SandboxDependencyCheck

  // 政策判定
  isAutoAllowBashIfSandboxedEnabled(): boolean
  areUnsandboxedCommandsAllowed(): boolean
  isSandboxRequired(): boolean
  areSandboxSettingsLockedByPolicy(): boolean

  // 设置
  setSandboxSettings(options: {...}): Promise<void>

  // 配置获取
  getFsReadConfig(): FsReadRestrictionConfig
  getFsWriteConfig(): FsWriteRestrictionConfig
  getNetworkRestrictionConfig(): NetworkRestrictionConfig
  getAllowUnixSockets(): string[] | undefined
  getAllowLocalBinding(): boolean | undefined
  getIgnoreViolations(): IgnoreViolationsConfig | undefined
  getEnableWeakerNestedSandbox(): boolean | undefined
  getExcludedCommands(): string[]
  getProxyPort(): number | undefined
  getSocksProxyPort(): number | undefined
  getLinuxHttpSocketPath(): string | undefined
  getLinuxSocksSocketPath(): string | undefined
  waitForNetworkInitialization(): Promise<boolean>

  // 执行 wrap
  wrapWithSandbox(
    command: string,
    binShell?: string,
    customConfig?: Partial<SandboxRuntimeConfig>,
    abortSignal?: AbortSignal,
  ): Promise<string>

  // 后处理
  cleanupAfterCommand(): void
  getSandboxViolationStore(): SandboxViolationStore
  annotateStderrWithSandboxFailures(command: string, stderr: string): string
  getLinuxGlobPatternWarnings(): string[]
  refreshConfig(): void
  reset(): Promise<void>
}
```

接口面很大——这是因为沙箱系统要支持：

- **多平台**：Darwin 用 sandbox-exec、Linux 用其他机制（landlock / seccomp / namespaces）；
- **多种限制类型**：fs read / fs write / network / unix socket / local binding；
- **运行时配置变化**：用户改 settings 后要 refresh config；
- **错误归因**：sandbox-exec 失败时把 violation 信息注入 stderr 让模型看到。

## 7.5 SandboxManager 实例 —— 适配器模式

`sandbox-adapter.ts:927-967`：

```ts
export const SandboxManager: ISandboxManager = {
  // Custom implementations
  initialize,
  isSandboxingEnabled,
  isSandboxEnabledInSettings: getSandboxEnabledSetting,
  isPlatformInEnabledList,
  getSandboxUnavailableReason,
  isAutoAllowBashIfSandboxedEnabled,
  areUnsandboxedCommandsAllowed,
  isSandboxRequired,
  areSandboxSettingsLockedByPolicy,
  setSandboxSettings,
  getExcludedCommands,
  wrapWithSandbox,
  refreshConfig,
  reset,
  checkDependencies,

  // Forward to base sandbox manager
  getFsReadConfig: BaseSandboxManager.getFsReadConfig,
  getFsWriteConfig: BaseSandboxManager.getFsWriteConfig,
  getNetworkRestrictionConfig: BaseSandboxManager.getNetworkRestrictionConfig,
  // ... 等等

  cleanupAfterCommand: (): void => {
    BaseSandboxManager.cleanupAfterCommand()
    scrubBareGitRepoFiles()
  },
}
```

`SandboxManager` 是 Claude Code 对 `BaseSandboxManager`（来自 sandbox-runtime 库）的**适配器**：

- **Custom implementations**：Claude CLI 特有的逻辑（如初始化时 detect worktree main repo path）；
- **Forward to base**：直接转发给底层 sandbox-runtime 的实现；
- **包装行为**：`cleanupAfterCommand` 在调底层后再加 `scrubBareGitRepoFiles()`。

这种适配器模式让 Claude Code 能复用 sandbox-runtime（可能是开源库），同时叠加 CLI 特定行为。

## 7.6 wrapWithSandbox —— 命令包装

`sandbox-adapter.ts:704-725`：

```ts
async function wrapWithSandbox(
  command: string,
  binShell?: string,
  customConfig?: Partial<SandboxRuntimeConfig>,
  abortSignal?: AbortSignal,
): Promise<string> {
  if (isSandboxingEnabled()) {
    if (initializationPromise) {
      await initializationPromise
    } else {
      throw new Error('Sandbox failed to initialize. ')
    }
  }

  return BaseSandboxManager.wrapWithSandbox(
    command,
    binShell,
    customConfig,
    abortSignal,
  )
}
```

简单地：**等初始化完成 → 调底层 wrapWithSandbox**。底层根据平台返回包装后的命令字符串。

例（Darwin 上的输出示意）：

```bash
# 输入: "ls /tmp"
# 输出大致:
sandbox-exec -f /tmp/sandbox-xxxx.sb /bin/bash -c "ls /tmp"
```

`sandbox-exec` 是 macOS 内置工具（虽然 Apple 已经 deprecated 但仍可用），`-f sandbox-profile.sb` 指定一个 sandbox profile（SBPL 语言写的策略）。profile 列出：

- `(deny default)` 默认拒绝
- `(allow file-read-data ...)` 允许读哪些路径
- `(allow file-write-data ...)` 允许写哪些路径
- `(allow network-outbound ...)` 允许出网到哪些主机
- 等等

profile 由 `convertToSandboxRuntimeConfig(settings)` 从用户设置生成。

## 7.7 sandbox profile 配置的来源

`sandbox-adapter.ts:898-906` 的 config getter：

```ts
getFsReadConfig(): FsReadRestrictionConfig
getFsWriteConfig(): FsWriteRestrictionConfig
getNetworkRestrictionConfig(): NetworkRestrictionConfig
getAllowUnixSockets(): string[] | undefined
getAllowLocalBinding(): boolean | undefined
getIgnoreViolations(): IgnoreViolationsConfig | undefined
```

每个 config 都从 `~/.claude/settings.json` 的 `sandbox` 字段读取，例：

```json
{
  "sandbox": {
    "enabled": true,
    "excludedCommands": ["bazel run", "make test"],
    "filesystem": {
      "read": {
        "denyOnly": ["/Users/*/.ssh", "/Users/*/.aws"],
        "allowWithinDeny": ["/Users/*/.config/claude-code"]
      },
      "write": {
        "allowOnly": ["$CLAUDE_PROJECT_DIR", "$TMPDIR"],
        "denyWithinAllow": ["**/node_modules/**/*.so"]
      }
    },
    "network": {
      "allowedHosts": ["github.com", "*.googleapis.com"],
      "deniedHosts": ["*.tracking.com"]
    }
  }
}
```

读限制用 **denyOnly**（黑名单）+ **allowWithinDeny**（黑名单里的白名单例外）。
写限制用 **allowOnly**（白名单）+ **denyWithinAllow**（白名单里的黑名单例外）。

这种**双向例外**让用户能写：
- "禁止读 ~/.ssh，但允许读 ~/.ssh/known_hosts"
- "允许写 ~/project，但禁止写 ~/project/node_modules/*.so"

非对称：read 是默认放、写是默认拒——符合直觉。

## 7.8 Initialize 路径 —— 异步且单例

`sandbox-adapter.ts:730-792`：

```ts
async function initialize(
  sandboxAskCallback?: SandboxAskCallback,
): Promise<void> {
  if (initializationPromise) {
    return initializationPromise  // 已经在初始化或已完成
  }
  if (!isSandboxingEnabled()) {
    return  // 用户没启用
  }

  // 包装 callback 强制 allowManagedDomainsOnly 政策
  const wrappedCallback: SandboxAskCallback | undefined = sandboxAskCallback
    ? async (hostPattern: NetworkHostPattern) => {
        if (shouldAllowManagedSandboxDomainsOnly()) {
          logForDebugging(`[sandbox] Blocked network request to ${hostPattern.host} ...`)
          return false
        }
        return sandboxAskCallback(hostPattern)
      }
    : undefined

  // 同步赋值 promise (在第一个 await 之前) - 防止 race condition
  initializationPromise = (async () => {
    try {
      if (worktreeMainRepoPath === undefined) {
        worktreeMainRepoPath = await detectWorktreeMainRepoPath(getCwdState())
      }

      const settings = getSettings_DEPRECATED()
      const runtimeConfig = convertToSandboxRuntimeConfig(settings)

      await BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)

      // 订阅 settings 变化, 动态更新 sandbox config
      settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
        const settings = getSettings_DEPRECATED()
        const newConfig = convertToSandboxRuntimeConfig(settings)
        BaseSandboxManager.updateConfig(newConfig)
      })
    } catch (error) {
      initializationPromise = undefined  // 失败时清理, 允许 retry
      logForDebugging(`Failed to initialize sandbox: ${errorMessage(error)}`)
    }
  })()

  return initializationPromise
}
```

3 个细节：

### 单例模式

`initializationPromise` 全局变量，第二次调用 `initialize()` 时直接返回已有 promise。

### Race condition 防御

```
// Create the initialization promise synchronously (before any await) to prevent
// race conditions where wrapWithSandbox() is called before the promise is assigned.
```

`wrapWithSandbox` 会 `await initializationPromise`——如果 `initialize` 在 await 之间被并发调用，第二个调用看到 promise 还没赋值。**先同步赋值再 await**——经典 JS 单例模式陷阱。

### 失败时清理允许 retry

```ts
} catch (error) {
  initializationPromise = undefined  // 清掉
  logForDebugging(`Failed to initialize sandbox: ${errorMessage(error)}`)
}
```

初始化失败 → 清 promise，让下次调用能重新尝试。否则一次失败永远卡住。

### settings 变化订阅

```ts
settingsSubscriptionCleanup = settingsChangeDetector.subscribe(() => {
  const newConfig = convertToSandboxRuntimeConfig(settings)
  BaseSandboxManager.updateConfig(newConfig)
})
```

用户改 settings.json → 自动 reload sandbox config。这避免**重启 Claude Code 才能让 sandbox 配置生效**。

## 7.9 annotateStderrWithSandboxFailures —— 违规归因

[01 工具入口](./01-工具入口与prompt定义.md) 提到 `BashTool.tsx:710`：

```ts
const outputWithSbFailures = SandboxManager.annotateStderrWithSandboxFailures(
  input.command, result.stdout || ''
)
```

这个方法做什么？沙箱 violation 通常表现为系统错误（`Operation not permitted` / `Permission denied` 等），但**不带具体原因**——模型看到 "Operation not permitted" 不知道是沙箱拦的还是真正的权限问题。

`annotateStderrWithSandboxFailures` 扫描 stderr，对照 sandbox violation store（运行时记录的违规事件），**在 stderr 里注入说明**：

```
Operation not permitted

[sandbox] Blocked file write to /etc/passwd (allowOnly does not include this path)
```

让模型看到 violation 详情 → 知道是沙箱拦的 → 可能建议 `dangerouslyDisableSandbox` 重试或换路径。

这是个**协议改善**——错误归因从 OS 级别迁移到应用层，让上层有更准确的信号。

## 7.10 nested sandbox —— `getEnableWeakerNestedSandbox`

`ISandboxManager.getEnableWeakerNestedSandbox()` 暗示**嵌套沙箱**的概念。

场景：Claude Code 自己在 sandbox 里跑（CI 容器、企业部署）→ 它再 spawn 一个 sandboxed bash → 内层 sandbox 被外层 sandbox 限制。

`enableWeakerNestedSandbox` 让内层沙箱放宽（避免双重限制导致正常命令失败）。

这是**真实部署场景驱动**的设计——不是设计阶段拍出来的。

## 7.11 cleanupAfterCommand 与 scrubBareGitRepoFiles

`sandbox-adapter.ts:963-966`：

```ts
cleanupAfterCommand: (): void => {
  BaseSandboxManager.cleanupAfterCommand()
  scrubBareGitRepoFiles()
},
```

每条命令执行完后 cleanup。Claude Code 额外做：`scrubBareGitRepoFiles()`。

这个函数（在 sandbox-adapter.ts 内部）清理 sandboxed 命令可能创建的 bare git repo 文件（hooks/HEAD/objects/refs）——防止 [05 path/readonly](./05-只读vs可写与路径校验.md) 的 bare repo 攻击的**事后清理**。

防御链上的位置：
- **前置**：[05] 检测 cd+git 复合 / bare repo / 写 git 内部路径 → ask
- **执行中**：sandbox profile 限制写路径
- **事后**：cleanupAfterCommand 清掉可能的残留

3 层保险——典型的纵深防御。

## 7.12 几个隐性设计判断

### 1. 沙箱不在乎"命令名"

permission 决策按"命令名"匹配规则，但沙箱按"行为"限制（路径 / 网络）。两者**正交**：

- permission 错放 `rm` → 沙箱仍拦写 `/`；
- permission 拦 `cat` → 沙箱无关；
- permission 允许 `npm install` → 沙箱仍拦写 system dirs。

正交意味着每一层独立改进、独立失败。

### 2. excludedCommands ≠ 安全边界

注释明确"NOTE: excludedCommands is a user-facing convenience feature, not a security boundary"。这种**显式 disclaimer** 在多年后的维护中防止误把它当 security 加固。

类似地，[04 permission 决策](./04-permission决策核心.md#bashpermissions-ts-705) 也明确 "Heuristic only — export-&& form bypasses this"——承认局限。

### 3. dangerouslyDisableSandbox 需要双重许可

模型设 `dangerouslyDisableSandbox: true` + 政策 `areUnsandboxedCommandsAllowed()` 都必须为真才跳过沙箱。**单方面授权不够**——一道是用户/管理员意愿，一道是模型当下判断。

### 4. 动态 config + settings 订阅

ant 内部用 GrowthBook 动态调 disabled commands；外部用户改 settings.json 自动 reload。两条路径让运维**不需要重启**就能调沙箱行为。

### 5. 同步赋值 promise 防 race

`initializationPromise = (async () => { ... })()` 先同步赋值，promise body 异步执行——经典 JS 单例模式陷阱的标准解。这种细节是工业 JS 代码的基本功。

### 6. 失败时 promise 清空允许 retry

`catch { initializationPromise = undefined }`——一次初始化失败不会永久卡住系统。沙箱初始化可能因为权限、依赖缺失等失败，要支持重试。

### 7. nested sandbox 的真实场景

`enableWeakerNestedSandbox` 是部署中真实出现的需求（容器里跑 Claude Code）。设计上**留口子**让真实场景能运行，但**默认不开**（保守）。

### 8. fail-graceful 而不是 throw

`initialize` 失败时只 log 不 throw：

> Log error but don't throw - let sandboxing fail gracefully

这是个设计判断：**沙箱失败时让命令仍能执行（不带沙箱）**而不是阻塞整个 CLI。理由：沙箱是辅助安全，不是核心功能；让 user 至少能用 Claude Code 比"安全启动失败"友好。

但这也意味着**用户必须自己监控沙箱是否真启用**——`SandboxManager.getSandboxUnavailableReason()` 让 UI 能显示这条信息。

## 7.13 与平台无关 vs 平台特有

ISandboxManager 接口在平台层是**抽象**的——只关心"配置 / 执行 wrap / cleanup"。具体实现（Darwin sandbox-exec / Linux landlock+seccomp / Windows ...）藏在 `BaseSandboxManager` 内部。

`isSupportedPlatform()` / `getSandboxUnavailableReason()` 让上层知道当前平台支持情况。Linux 有自己的 quirks（`getLinuxGlobPatternWarnings()` / `getLinuxHttpSocketPath()` 等）——接口里加平台特定 method。

## 7.14 小结

- 沙箱是闸门 4——行为面收紧，不在乎命令名，OS 级隔离；
- `shouldUseSandbox` 4 步：整体启用 → dangerouslyDisableSandbox + 政策 → 有命令 → excluded;
- excluded 检查走 fixed-point 迭代处理多层 wrapper + env var；
- excluded 明确**不是安全边界**——是用户便利特性，注释 disclaimer；
- ISandboxManager 接口 30+ 方法支持多平台、多种限制、运行时 reload；
- Darwin 用 sandbox-exec + SBPL profile；Linux 有自己的机制；
- read 用 denyOnly + allowWithinDeny（默认放），write 用 allowOnly + denyWithinAllow（默认拒）——非对称；
- 初始化用 synchronous-assign promise 防 race；失败清空允许 retry；
- `annotateStderrWithSandboxFailures` 把 OS 级错误归因到沙箱 violation，让模型有可读信号；
- `cleanupAfterCommand + scrubBareGitRepoFiles` 是 bare repo 攻击的事后清理；
- 设计上沙箱**fail-graceful** 不阻塞 CLI 启动——但通过 UI 暴露状态让用户知情。

下一篇 → [08 执行层与环境](./08-执行层与环境.md)
