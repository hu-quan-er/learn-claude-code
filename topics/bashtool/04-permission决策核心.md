# 04 permission 决策核心

> `bashPermissions.ts` 2621 行——`bashToolHasPermission` / `bashToolCheckPermission` 是核心决策引擎。本篇拆它的 8 步决策树、wildcard 匹配、env var / wrapper 透视、speculative classifier 预判机制。

## 4.1 8 步决策树

`bashToolCheckPermission`（`bashPermissions.ts:1050+`）是决策中枢。简化的伪代码：

```ts
export const bashToolCheckPermission = (input, toolPermissionContext, ...) => {
  // 1. 精确匹配
  const exactMatchResult = bashToolCheckExactMatchPermission(input, ctx)
  if (exactMatchResult.behavior in ['deny', 'ask']) return exactMatchResult

  // 2. 前缀匹配 deny/ask 规则
  const { matchingDenyRules, matchingAskRules, matchingAllowRules } =
    matchingRulesForInput(input, ctx, 'prefix', { ... })

  // 2a. 命中 deny
  if (matchingDenyRules[0]) {
    return { behavior: 'deny', message: `Permission... denied.`, decisionReason: { type: 'rule', rule: matchingDenyRules[0] } }
  }
  // 2b. 命中 ask
  if (matchingAskRules[0]) {
    return { behavior: 'ask', message: ..., decisionReason: { type: 'rule', rule: matchingAskRules[0] } }
  }

  // 3. 路径约束检查（pathValidation）
  const pathResult = checkPathConstraints(input, getCwd(), ctx, compoundCommandHasCd, astCommand?.redirects, ...)
  if (pathResult.behavior !== 'passthrough') return pathResult

  // 4. allow 精确匹配
  if (exactMatchResult.behavior === 'allow') return exactMatchResult

  // 5. allow 前缀匹配
  if (matchingAllowRules[0]) {
    return { behavior: 'allow', updatedInput: input, decisionReason: { type: 'rule', rule: matchingAllowRules[0] } }
  }

  // 5b. sed 约束
  const sedConstraintResult = checkSedConstraints(input, ctx)
  if (sedConstraintResult.behavior !== 'passthrough') return sedConstraintResult

  // 6. 权限模式（auto/yolo/plan/...）专属逻辑
  const modeResult = checkPermissionMode(input, ctx)
  if (modeResult.behavior !== 'passthrough') return modeResult

  // 7. 只读命令默认 allow
  if (BashTool.isReadOnly(input)) {
    return { behavior: 'allow', updatedInput: input, decisionReason: { type: 'other', reason: 'Read-only command is allowed' } }
  }

  // 8. 没有规则匹配 → passthrough（触发用户确认）
  return {
    behavior: 'passthrough',
    message: createPermissionRequestMessage(BashTool.name, decisionReason),
    decisionReason: { type: 'other', reason: 'This command requires approval' },
    suggestions: suggestionForExactCommand(command),
  }
}
```

**决策顺序的精心设计**：

- **deny 在 allow 之前**：用户禁止过的命令永远 deny，即使有 allow 规则；
- **path 检查在 allow 之前**：危险路径访问总要拦，无论用户是否 allow 过；
- **sed 约束在 mode 之前**：sed 模拟执行（[06](./06-sed与wrapper-specs.md)）走专门通道；
- **只读默认 allow 在末尾**：先看显式规则、再看模式、最后兜底"看起来安全的就 allow"。

注释（`bashPermissions.ts:1073-1074`）：

> SECURITY FIX: Check Bash deny/ask rules BEFORE path constraints to prevent bypass via absolute paths outside the project directory (HackerOne report)

明确：这是一个**修过的漏洞**——之前 path constraint 在前，用户能用 `/etc/...` 绝对路径绕过 deny 规则。修复后 deny 总是最高优先级。

## 4.2 PermissionRule 与 decisionReason

每条规则匹配后返回的 `PermissionResult` 都带 `decisionReason`：

```ts
decisionReason: {
  type: 'rule',
  rule: matchingDenyRules[0],
}

// 或
decisionReason: {
  type: 'other',
  reason: 'Read-only command is allowed',
}
```

这是 [core-models 05 Permission](../core-models/05-Permission-权限模型.md) 讲的"11 reasons"中的两条。`decisionReason` 让 UI 能精确显示"为什么这条命令被允许/拒绝"——比仅返回 boolean 友好得多。

## 4.3 wildcard 匹配 —— `Bash(git diff:*)` 规则

`matchWildcardPattern`（`bashPermissions.ts:353-358`）：

```ts
export function matchWildcardPattern(
  pattern: string,
  command: string,
): boolean {
  return sharedMatchWildcardPattern(pattern, command)
}
```

把工作委托给 shared 实现。规则字面值如 `Bash(git diff:*)` 意思是：

- `git diff` —— 精确匹配 `git diff` 完整命令；
- `git diff:*` —— 前缀匹配，包括 `git diff foo.ts`、`git diff --cached` 等任何以 `git diff ` 开头的。

实际匹配前要先**透视** wrapper / env var：

- 输入命令：`timeout 30 FOO=bar git diff foo.ts`
- 透视后：`git diff foo.ts`
- 匹配规则：`Bash(git diff:*)` → 命中

这种"看穿外壳后再匹配"是后面 4.4-4.5 的核心机制。

## 4.4 SAFE_ENV_VARS —— 安全环境变量白名单

`bashPermissions.ts:378-430` 是个长名单。**白名单**意味着只列**已知绝对安全**的——其它任何环境变量都不被剥离，让权限规则按"带前缀"的字符串匹配（更严）。

注释（`bashPermissions.ts:371-377`）：

> SECURITY: These must NEVER be added to the whitelist:
> - PATH, LD_PRELOAD, LD_LIBRARY_PATH, DYLD_* (execution/library loading)
> - PYTHONPATH, NODE_PATH, CLASSPATH, RUBYLIB (module loading)
> - GOFLAGS, RUSTFLAGS, NODE_OPTIONS (can contain code execution flags)
> - HOME, TMPDIR, SHELL, BASH_ENV (affect system behavior)

明确列出"**绝不能加进白名单**"的环境变量类别——动态库加载、模块加载、编译器 flag、系统行为。

SAFE_ENV_VARS 实际包含的（精简）：

- Go：`GOEXPERIMENT` / `GOOS` / `GOARCH` / `CGO_ENABLED` / `GO111MODULE`
- Rust：`RUST_BACKTRACE` / `RUST_LOG`
- Node：`NODE_ENV`（**不**包含 `NODE_OPTIONS`）
- Python：`PYTHONUNBUFFERED` / `PYTHONDONTWRITEBYTECODE`（**不**包含 `PYTHONPATH`）
- Pytest：`PYTEST_DISABLE_PLUGIN_AUTOLOAD` / `PYTEST_DEBUG`
- API key：`ANTHROPIC_API_KEY`
- 区域：`LANG` / `LANGUAGE` / `LC_*` / `CHARSET`
- 终端：`TERM` / `COLORTERM` / `NO_COLOR` / `FORCE_COLOR` / `TZ`
- 颜色配置：`LS_COLORS` / `LSCOLORS` / `GREP_COLORS` / `GCC_COLORS` 等
- 显示格式：`TIME_STYLE` / `BLOCK_SIZE`

注意：**没有 `PATH`**——`PATH=xxx cmd` 会被当不安全前缀保留，让 deny 规则像 `Bash(PATH=*)` 之类的能匹配。

## 4.5 stripSafeWrappers —— 透视器命令剥离

`bashPermissions.ts:524-615`。这个函数把 `timeout 30 X` 透视成 `X`。

支持的 wrapper：

```ts
const SAFE_WRAPPER_PATTERNS = [
  /^timeout[ \t]+...复杂的 timeout flag 处理.../,
  /^time[ \t]+(?:--[ \t]+)?/,
  /^nice(?:[ \t]+-n[ \t]+-?\d+|[ \t]+-\d+)?[ \t]+(?:--[ \t]+)?/,
  /^stdbuf(?:[ \t]+-[ioe][LN0-9]+)+[ \t]+(?:--[ \t]+)?/,
  /^nohup[ \t]+(?:--[ \t]+)?/,
]
```

5 个 wrapper：timeout / time / nice / stdbuf / nohup。

### 关键安全细节 1：`[ \t]+` 不是 `\s+`

```
SECURITY: Use [ \t]+ not \s+ — \s matches \n/\r which are command separators in bash.
```

`\s` 会匹配换行/回车，但 bash 把换行当**命令分隔符**。如果用 `\s+`：

```
nohup\nrm -rf /
```

会被"剥掉 nohup 加换行"变成 `rm -rf /`——但 bash 实际看到两段：`nohup`（自己） + `rm -rf /`（另一段）。两边对命令的理解不同 → 漏检 `rm -rf /`。

修复：只用水平空白 `[ \t]+`，让换行保留为分隔符。

### 关键安全细节 2：`(?:--[ \t]+)?`

```
SECURITY: `(?:--[ \t]+)?` consumes the wrapper's own `--` so
`nohup -- rm -- -/../foo` strips to `rm -- -/../foo` (not `-- rm ...`
which would skip path validation with `--` as an unknown baseCmd).
```

`--` 是 wrapper 的"参数终止符"——区分 wrapper 的选项和它后面的命令。剥 wrapper 时要同时剥掉它的 `--`，否则下游会看到 `-- rm ...` 把 `--` 当 baseCmd（未知命令 → 跳过 path 检查）。

### 关键安全细节 3：timeout 的 flag 值用 allowlist

`bashPermissions.ts:617-620`：

```ts
// SECURITY: allowlist for timeout flag VALUES (signals are TERM/KILL/9,
// durations are 5/5s/10.5). Rejects $ ( ) ` | ; & and newlines that
// previously matched via [^ \t]+ — `timeout -k$(id) 10 ls` must NOT strip.
const TIMEOUT_FLAG_VALUE_RE = /^[A-Za-z0-9_.+-]+$/
```

历史 bug：之前 `--signal` 的值用 `[^ \t]+`（非空白） → 攻击者写：

```bash
timeout -k$(id) 10 ls
```

`$(id)` 被当 `-k` 的值剥掉 → 看似只剩 `ls` → 误以为安全。但 bash 在 timeout 运行**之前**就展开 `$(id)`，整个进程图都被劫持。

修复：flag 值只允许 `[A-Za-z0-9_.+-]`——拒绝 shell 元字符。

### 关键安全细节 4：两阶段剥离不混合

```ts
// Phase 1: Strip leading env vars and comments only.
// Phase 2: Strip wrapper commands and comments only. Do NOT strip env vars.
// Wrapper commands (timeout, time, nice, nohup) use execvp to run their
// arguments, so VAR=val after a wrapper is treated as the COMMAND to execute.
// (HackerOne #3543050)
```

**两阶段分开**：
- Phase 1 剥 leading env vars（在任何命令前的）；
- Phase 2 剥 wrapper（剥完不再回去剥 env var）；

为什么？因为 **wrapper 后面的 `VAR=val` 不是 shell-level 赋值**——`timeout 30 FOO=bar` 把 `FOO=bar` 当作要执行的命令（execvp 把整个 argv 传给 timeout，timeout 内部用 execvp 启动 `FOO=bar`，找不到这个二进制 → 报错）。

如果继续剥 env var，会以为 `FOO=bar git push` 是合法形式，但实际上 wrapper 不认这种结构。HackerOne #3543050 报告了这条。

## 4.6 stripAllLeadingEnvVars —— deny 规则用的"激进剥离"

`bashPermissions.ts:733-776`。和 `stripSafeWrappers` 的 env var 阶段不同——这个函数**剥所有 env var**，不只是白名单里的。

为什么有两套？注释（`bashPermissions.ts:710-721`）：

> Used for deny/ask rule matching: when a user denies `claude` or `rm`, the command should stay blocked even if prefixed with arbitrary env vars like `FOO=bar claude`. The safe-list restriction in stripSafeWrappers is correct for allow rules (prevents `DOCKER_HOST=evil docker ps` from auto-matching `Bash(docker ps:*)`), but **deny rules must be harder to circumvent**.

不对称：
- **allow 规则**用 `stripSafeWrappers`（白名单）——只剥已知安全的 env var，"不被认识的"保留让规则更严；
- **deny 规则**用 `stripAllLeadingEnvVars`（黑名单）——剥所有 env var（除 BINARY_HIJACK 类），让 `FOO=bar denied_cmd` 仍能被拦。

allow 严格 / deny 宽松——这是个非对称的安全设计：**给攻击者更少漏洞，给用户更多透明度**。

## 4.7 BINARY_HIJACK_VARS —— 防动态链接劫持

`bashPermissions.ts:708`：

```ts
export const BINARY_HIJACK_VARS = /^(LD_|DYLD_|PATH$)/
```

这一行短短的 regex 是整个 permission 系统里最重要的几行之一。监控 3 类环境变量：

| 类别 | 例 | 危害 |
|------|---|------|
| `LD_*` | `LD_PRELOAD=/tmp/evil.so` / `LD_LIBRARY_PATH=/tmp/` | Linux 动态链接器：把任意 .so 插进所有动态链接的进程 |
| `DYLD_*` | `DYLD_INSERT_LIBRARIES=/tmp/evil.dylib` / `DYLD_FALLBACK_LIBRARY_PATH=/tmp/` | macOS 同上 |
| `PATH` | `PATH=/tmp:$PATH npm install` | 让某个命令解析到攻击者的二进制 |

`BINARY_HIJACK_VARS` 被用在两处：

1. `stripAllLeadingEnvVars` 的 blocklist 参数——遇到这类 var **停止剥离**，让规则按"带前缀"形式匹配（更严）；
2. `shouldUseSandbox`（[07](./07-Sandbox沙箱机制.md)）的 excludedCommands 匹配——这类 var 前缀的命令必须进 sandbox。

注释（`bashPermissions.ts:703-708`）：

> Env vars that make a *different binary* run (injection or resolution hijack). Heuristic only — export-&& form bypasses this, and excludedCommands isn't a security boundary anyway.

注释承认：`export LD_PRELOAD=xxx && cmd` 这种形式绕过——因为 `&&` 把命令拆成两段，第二段没有前缀。但 BashTool 会把 `&&` 拆开后逐段检查（[02 命令解析管线](./02-命令解析管线.md)），所以 `export` 那一段也会单独走规则。同时 sandbox 不是 BINARY_HIJACK 的主防御——只是辅助。

## 4.8 ANT_ONLY_SAFE_ENV_VARS —— ant 内部专属白名单

`bashPermissions.ts:432-` 后面有个 `ANT_ONLY_SAFE_ENV_VARS`，注释：

> ANT-ONLY environment variables that are safe to strip from commands. These are only enabled when USER_TYPE === 'ant'.
>
> SECURITY: These env vars are stripped before permission-rule matching, which means `DOCKER_HOST=tcp://evil.com docker ps` matches a `Bash(docker ps:*)` rule after stripping. This is INTENTIONALLY ANT-ONLY (gated at line ~380) and MUST NEVER ship to external users.

`DOCKER_HOST` / `KUBECONFIG` 这些环境变量在 ant 内部环境是常见的（指向 staging cluster 等），剥离它们让规则匹配更顺。但**对外部用户**有真实风险（`DOCKER_HOST=evil` 让 docker 命令连到攻击者控制的 daemon）。

这种"内部宽松、外部严格"的双轨设计——通过 `process.env.USER_TYPE === 'ant'` 编译时常量控制 dead-code elimination，确保**外部 build 完全不包含**这条放宽路径。

## 4.9 路径约束检查 —— 委托 pathValidation

`bashPermissions.ts:1106-1122`：

```ts
const pathResult = checkPathConstraints(
  input,
  getCwd(),
  toolPermissionContext,
  compoundCommandHasCd,
  astCommand?.redirects,
  astCommand ? [astCommand] : undefined,
)
if (pathResult.behavior !== 'passthrough') {
  return pathResult
}
```

委托到 `pathValidation.ts`（1303 行，[05](./05-只读vs可写与路径校验.md) 详述）。一个关键点：**如果有 AST 解析结果，优先用它**。注释（`bashPermissions.ts:1108-1111`）：

> SECURITY: When AST-derived argv is available for this subcommand, pass it through so checkPathConstraints uses it directly instead of re-parsing with shell-quote (which has a single-quote backslash bug that causes parseCommandArguments to return [] and silently skip path validation).

shell-quote 库有个已知 bug：单引号里的 backslash 处理错误 → `parseCommandArguments` 返回空数组 → 路径校验**静默跳过**。AST-derived argv 没这个 bug。所以传 argv 进去强制走精确路径。

## 4.10 speculative classifier —— auto-mode 性能优化

`bashPermissions.ts:1491-1546` 的 `peekSpeculativeClassifierCheck` / `startSpeculativeClassifierCheck` / `consumeSpeculativeClassifierCheck` 实现了一个**预测性优化**：

**问题**：auto-mode 下用一个小模型 classifier 判断"这条命令是否安全可自动执行"。classifier 调用要 ~500ms 网络往返——会延迟工具执行。

**优化**：模型还在流式生成 assistant message 时，**已经看到了 tool_use 块的 input** → 提前启动 classifier。等 message 结束、要真正决策时，classifier 结果可能已经回来了。

```ts
// 启动期（assistant message 流式中，tool_use 块已可见）
startSpeculativeClassifierCheck(command, ctx, signal, isNonInteractiveSession)
  → 把 classifier promise 存到 speculativeChecks: Map<command, Promise>

// 决策期（即将执行工具，要权限决策）
const pending = peekSpeculativeClassifierCheck(command)
if (pending) {
  // 复用已经在飞的 classifier 调用
  result = await pending
} else {
  // 没有 speculative → 现场启动 + await
  result = await classifyBashCommand(...)
}
```

这是个**很巧妙的并发优化**——把"等待"叠加到"模型流式"上，对用户看起来 latency 是 0。

注意几个边界处理：

```ts
// 防止 unhandled rejection
promise.catch(() => {})
speculativeChecks.set(command, promise)
```

speculative promise 可能被 abort，但即使 reject 也要 `.catch(()=>{})` 防止 unhandled——因为消费者可能根本不来 await。

```ts
// gating
if (!isClassifierPermissionsEnabled()) return false
if (feature('TRANSCRIPT_CLASSIFIER') && ctx.mode === 'auto') return false
if (ctx.mode === 'bypassPermissions') return false
const allowDescriptions = getBashPromptAllowDescriptions(ctx)
if (allowDescriptions.length === 0) return false
```

多重 gating：feature flag、permission mode、是否有 prompt 形式的允许描述。

## 4.11 bashToolHasPermission —— 顶层包装

注意 BashTool.tsx 调用的是 `bashToolHasPermission`，不是 `bashToolCheckPermission`。差别？前者是顶层包装，对复合命令做拆分 + 多次调用。

伪代码：

```ts
async function bashToolHasPermission(input, context) {
  const command = input.command.trim()

  // 1. 解析命令 (AST 优先)
  const parsed = await parseForSecurity(command)

  if (parsed.kind === 'too-complex') {
    return { behavior: 'ask', message: parsed.reason, decisionReason: ... }
  }

  if (parsed.kind === 'simple') {
    // 2. 复合命令: 逐 subcommand 检查
    for (const subcommand of parsed.commands) {
      const subResult = bashToolCheckPermission(
        { command: reconstructCommand(subcommand) },
        context,
        compoundCommandHasCd,
        subcommand,
      )
      if (subResult.behavior === 'deny') return subResult  // 任一 deny 即整体 deny
      if (subResult.behavior === 'ask')  return subResult  // 任一 ask 即整体 ask
      // allow / passthrough 继续
    }
    return { behavior: 'allow', ... }
  }

  if (parsed.kind === 'parse-unavailable') {
    // 走 legacy 路径 (regex-based)
    return bashCommandIsSafeAsync(command).then(...)
  }
}
```

实际代码比这复杂得多（处理 sed / classifier / 复合命令 cd 状态等），但核心结构是这样。

## 4.12 几个隐性设计判断

### 1. deny 永远在 allow 之前

经典安全设计：**禁止覆盖允许**。即使用户某场景明确 `Bash(curl:*)` 允许了 curl，遇到 `Bash(curl evil.com)` 这类精确 deny 仍要拦。

### 2. allow / deny 的剥离对称性不同

allow 用白名单（严格）、deny 用黑名单（宽松）。这种"非对称剥离"在安全设计里是常见的"对正常用户友好、对攻击者严格"思路。

### 3. AST 优先于 regex

bashToolCheckPermission 接受可选的 `astCommand: SimpleCommand` 参数——有 AST 就用 AST，没 AST 才回退到 regex/shell-quote。注释明确说明 shell-quote 有 single-quote backslash bug 导致路径校验跳过——这是一个真实的 silently-fail 漏洞。

### 4. wrapper 透视支持 5 个命令

只支持 `timeout` / `time` / `nice` / `stdbuf` / `nohup`——保守。其它"看起来像 wrapper"的命令（如 `xargs` / `env` / `setsid`）都不透视，让规则按"看到的命令名"匹配。

加新 wrapper 是个高风险动作——每加一个就要确认它没有自己的攻击向量（如 `xargs` 的 `-I {}` 可以构造任意命令）。

### 5. fixed-point 迭代处理混合包装

`filterRulesByContentsMatchingInput` 里：

```ts
const seen = new Set(commandsToTry)
let startIdx = 0
while (startIdx < commandsToTry.length) {
  const endIdx = commandsToTry.length
  for (let i = startIdx; i < endIdx; i++) {
    // 既试 stripAllLeadingEnvVars，又试 stripSafeWrappers
    // 直到没有新候选产生
  }
  startIdx = endIdx
}
```

**迭代直到 fixed-point**——处理 `nohup FOO=bar timeout 5 claude` 这类层叠包装。一次只剥一层会漏。注释（`bashPermissions.ts:817-825`）：

> Without iteration, single-pass compositions miss multi-layer interleaving.

### 6. speculative classifier 是延迟优化的范例

把"该等待的事"叠加到"反正在等待的事"上——经典的并发优化。设计要点：

- 状态机管理（startSpeculative → peekSpeculative → consumeSpeculative）；
- abort 信号传播；
- unhandled rejection 防御；
- 多重 gating（feature flag / mode / config）。

### 7. HackerOne 引用 ≥ 3 处

- `#21503` —— stripWrappersFromArgv vs checkSemantics 的 nice wrapper 不对称
- `#3543050` —— wrapper 后面的 VAR=val 不是 env var
- HackerOne 未具体编号的"BLU bash deny bypass via absolute paths"

每个修复都在注释里留下编号——保留**漏洞历史**让未来工程师能查到完整背景。这种"漏洞编号即代码注释"是这套项目的工程文化。

## 4.13 与 [prompt-injection 专题] 的对照

回顾 [prompt-injection 06 端到端模拟](../prompt-injection/06-元防御指令与端到端模拟.md) 的 6 层防御：

| Layer | prompt-injection 专题对应 | 本专题对应 |
|-------|------------------------|------------|
| Layer 1 字符级 | Unicode 清洗 | bashSecurity 的 control char / Unicode whitespace 检查 |
| Layer 2 格式 | FileRead 行号前缀 | AST 解析 + 复合命令拆分 |
| Layer 3 标签 | `<system-reminder>` 标签 | （此处不适用 BashTool） |
| Layer 4 预算 | attachments 5×4KB | （此处不适用） |
| Layer 5 指令层 | 系统提示元指令 | BashTool prompt 里的"优先专用工具" |
| Layer 6 行为护栏 | malware reminder | bashToolCheckPermission 的 deny 决策 |

加上**动作侧的 4 层**（解析 / security / permission / sandbox），覆盖远超 prompt-injection 专题。两个专题的关系是 **"被骗前防御 + 被骗后拦截"**——一前一后形成完整闭环。

## 4.14 小结

- `bashToolCheckPermission` 8 步决策树：精确 → 前缀 deny/ask → 路径 → allow → sed → 模式 → 只读 → 默认 ask；
- deny 总在 allow 之前，path 在 allow 之前——这是修过的漏洞顺序；
- allow 用白名单剥 env var（严格）、deny 用黑名单剥（宽松）——非对称设计；
- `stripSafeWrappers` 透视 5 个 wrapper（timeout/time/nice/stdbuf/nohup），多个安全细节（`[ \t]+` vs `\s+`、`--` 终止符、flag 值 allowlist）；
- `BINARY_HIJACK_VARS = /^(LD_|DYLD_|PATH$)/` 是动态链接劫持的最后一道防线；
- ANT_ONLY 白名单走编译期 dead-code elimination，外部 build 完全不包含；
- AST-derived argv 优先于 shell-quote re-parse（后者有 single-quote backslash bug）；
- speculative classifier 把模型流式时间叠加到 classifier 等待——并发优化范例；
- fixed-point 迭代处理多层 wrapper + env var 包装；
- HackerOne 引用是这套项目的工程文化标志。

下一篇 → [05 只读 vs 可写 + 路径校验](./05-只读vs可写与路径校验.md)
