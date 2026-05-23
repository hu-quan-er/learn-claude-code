# 03 AST 级 security 检查

> `bashSecurity.ts` 2592 行——23 个独立 validator，每个对应一类已知攻击模式。本篇拆它的架构、关键 validator 的实现思路、以及为什么需要这么多独立检查。

## 3.1 整体架构：ValidationContext + N 个 validator

`bashSecurity.ts` 的结构很规整：

```ts
type ValidationContext = {
  originalCommand: string
  baseCommand: string
  unquotedContent: string
  fullyUnquotedContent: string
  fullyUnquotedPreStrip: string
  unquotedKeepQuoteChars: string
  treeSitter?: TreeSitterAnalysis | null
}

function validateXxx(ctx: ValidationContext): PermissionResult { ... }
function validateYyy(ctx: ValidationContext): PermissionResult { ... }
...
```

每个 validator：
- 输入：context（含原命令、各种 unquote 形式、tree-sitter 分析）；
- 输出：`PermissionResult`，三选一：
  - `{ behavior: 'ask', message }` → 中止链路，弹用户确认；
  - `{ behavior: 'deny', message }` → 中止链路，拒绝；
  - `{ behavior: 'passthrough', message }` → 放过，由下一 validator 继续；

链路上**任一 validator 命中 ask/deny 就中止**——passthrough 模式。这种"短路链" 让每条规则独立、可测、可加。

## 3.2 BASH_SECURITY_CHECK_IDS —— 23 个 checkId 枚举

`bashSecurity.ts:77-101` 把每条 check 编号：

```ts
const BASH_SECURITY_CHECK_IDS = {
  INCOMPLETE_COMMANDS: 1,
  JQ_SYSTEM_FUNCTION: 2,
  JQ_FILE_ARGUMENTS: 3,
  OBFUSCATED_FLAGS: 4,
  SHELL_METACHARACTERS: 5,
  DANGEROUS_VARIABLES: 6,
  NEWLINES: 7,
  DANGEROUS_PATTERNS_COMMAND_SUBSTITUTION: 8,
  DANGEROUS_PATTERNS_INPUT_REDIRECTION: 9,
  DANGEROUS_PATTERNS_OUTPUT_REDIRECTION: 10,
  IFS_INJECTION: 11,
  GIT_COMMIT_SUBSTITUTION: 12,
  PROC_ENVIRON_ACCESS: 13,
  MALFORMED_TOKEN_INJECTION: 14,
  BACKSLASH_ESCAPED_WHITESPACE: 15,
  BRACE_EXPANSION: 16,
  CONTROL_CHARACTERS: 17,
  UNICODE_WHITESPACE: 18,
  MID_WORD_HASH: 19,
  ZSH_DANGEROUS_COMMANDS: 20,
  BACKSLASH_ESCAPED_OPERATORS: 21,
  COMMENT_QUOTE_DESYNC: 22,
  QUOTED_NEWLINE: 23,
} as const
```

每条 check 命中时上报 analytics：

```ts
logEvent('tengu_bash_security_check_triggered', {
  checkId: BASH_SECURITY_CHECK_IDS.IFS_INJECTION,
  subId: 1,
})
```

**为什么用 numeric ID 而不是字符串名**？注释（76）说 "to avoid logging strings"——避免把 checkId 字符串本身打进 analytics（隐私 + 大小考虑）。这种细节体现可观测性设计的成熟度。

23 类 check 可以分 6 组：

| 组 | check | 防御什么 |
|---|---|---|
| **不完整/非法语法** | 1, 14 | incomplete command / malformed token injection |
| **字符级注入** | 7, 15, 16, 17, 18, 21, 22, 23 | newline / backslash whitespace / brace expansion / 控制字符 / Unicode 空白 / backslash operators / comment quote desync / quoted newline |
| **环境变量攻击** | 6, 11 | dangerous vars (LD_PRELOAD 等) / IFS injection |
| **命令替换/重定向** | 8, 9, 10 | $() / process substitution / 重定向到危险目标 |
| **shell 元字符与混淆** | 4, 5, 19 | obfuscated flags / shell metachar / 单词内 `#` |
| **特定命令检查** | 2, 3, 12, 13, 20 | jq system function / jq file args / git commit / /proc/environ / zsh-only commands |

下面挑几个最有代表性的展开。

## 3.3 validateIFSInjection —— IFS 攻击

`bashSecurity.ts:1017-1036`：

```ts
function validateIFSInjection(context: ValidationContext): PermissionResult {
  const { originalCommand } = context

  if (/\$IFS|\$\{[^}]*IFS/.test(originalCommand)) {
    logEvent('tengu_bash_security_check_triggered', {
      checkId: BASH_SECURITY_CHECK_IDS.IFS_INJECTION,
      subId: 1,
    })
    return {
      behavior: 'ask',
      message: 'Command contains IFS variable usage which could bypass security validation',
    }
  }

  return { behavior: 'passthrough', message: 'No IFS injection detected' }
}
```

很短，但是经典攻击向量。

`$IFS` 是 bash 内置的"词分隔符"环境变量，默认是 `<space><tab><newline>`。攻击者可以用它构造**字面上看不出问题但执行时危险**的命令：

```bash
cat$IFS/etc/passwd
```

字面看，`cat$IFS/etc/passwd` 是一个 token——但因为 `$IFS` 被展开成空白，bash 实际执行 `cat /etc/passwd`——两个 token。

更狠的：

```bash
${IFS}cat${IFS}/etc/passwd
```

regex 找 "cat" 找不到（中间被 `${IFS}` 隔开），但 bash 执行起来一样是 `cat /etc/passwd`。

正则 `/\$IFS|\$\{[^}]*IFS/` 捕获两种形式：
- `$IFS` 直接引用；
- `${...IFS...}` 参数展开里含 IFS（如 `${IFS:0:1}` 只取一个字符）。

命中 → ask user。**不直接 deny**——`$IFS` 在合法脚本里也可能出现（很少但有），让用户判断。

## 3.4 validateProcEnvironAccess —— /proc 环境变量读取

`bashSecurity.ts:1041-1067`：

```ts
function validateProcEnvironAccess(context: ValidationContext): PermissionResult {
  const { originalCommand } = context

  if (/\/proc\/.*\/environ/.test(originalCommand)) {
    logEvent('tengu_bash_security_check_triggered', {
      checkId: BASH_SECURITY_CHECK_IDS.PROC_ENVIRON_ACCESS,
      subId: 1,
    })
    return {
      behavior: 'ask',
      message: 'Command accesses /proc/*/environ which could expose sensitive environment variables',
    }
  }

  return { behavior: 'passthrough', message: 'No /proc/environ access detected' }
}
```

Linux 上 `/proc/<pid>/environ` 暴露了该进程的环境变量。如果当前进程有敏感的 API key 在环境变量里：

```bash
cat /proc/self/environ
cat /proc/1/environ
find /proc -name environ -exec cat {} \;
```

都能读出来。

注释（`bashSecurity.ts:1038-1040`）：

> Additional hardening against reading environment variables via /proc filesystem. Path validation typically blocks /proc access, but this provides defense-in-depth.

明确说这是 **defense-in-depth**——路径校验（`pathValidation.ts`，详见 [05](./05-只读vs可写与路径校验.md)）已经会拦 /proc，但这里再加一道独立的字符串级检查。

## 3.5 validateMalformedTokenInjection —— eval 重解析旁路

`bashSecurity.ts:1069-1128`：

```ts
function validateMalformedTokenInjection(context: ValidationContext): PermissionResult {
  const { originalCommand } = context

  const parseResult = tryParseShellCommand(originalCommand)
  if (!parseResult.success) {
    return { behavior: 'passthrough', message: 'Parse failed, handled elsewhere' }
  }

  const parsed = parseResult.tokens
  const hasCommandSeparator = parsed.some(
    entry => typeof entry === 'object' && entry !== null && 'op' in entry &&
      (entry.op === ';' || entry.op === '&&' || entry.op === '||'),
  )

  if (!hasCommandSeparator) {
    return { behavior: 'passthrough', message: 'No command separators' }
  }

  if (hasMalformedTokens(originalCommand, parsed)) {
    logEvent('tengu_bash_security_check_triggered', {
      checkId: BASH_SECURITY_CHECK_IDS.MALFORMED_TOKEN_INJECTION,
      subId: 1,
    })
    return {
      behavior: 'ask',
      message: 'Command contains ambiguous syntax with command separators that could be misinterpreted',
    }
  }

  return { behavior: 'passthrough', message: 'No malformed token injection detected' }
}
```

注释（`bashSecurity.ts:1069-1080`）说得很具体：

> Detects commands with malformed tokens (unbalanced delimiters) combined with command separators. This catches potential injection patterns where ambiguous shell syntax could be exploited.
>
> **Security: This check catches the eval bypass discovered in HackerOne review.** When shell-quote parses ambiguous patterns like `echo {"hi":"hi;evil"}`, it may produce unbalanced tokens (e.g., `{hi:"hi`). Combined with command separators, this can lead to unintended command execution via eval re-parsing.

具体攻击：

```bash
echo {"hi":"hi;evil"}
```

- shell-quote 解析后可能产生 unbalanced token（如 `{hi:"hi`）；
- 加上 `;` / `&&` / `||` 分隔符；
- 下游某些 eval 路径重新解析时可能错误地把它当多个命令。

**这是一个 HackerOne 报告过的真实漏洞**。修复方式：检测到"malformed token + 命令分隔符" → ask user。

注意逻辑顺序：先 check `hasCommandSeparator` 再 check `hasMalformedTokens`——malformed token **本身**可能是 false positive（语法异常但无害），加上分隔符才升级为攻击向量。

## 3.6 ZSH_DANGEROUS_COMMANDS —— zsh 专有攻击面

`bashSecurity.ts:45-74` 是个长清单：

```ts
const ZSH_DANGEROUS_COMMANDS = new Set([
  // zmodload is the gateway to many dangerous module-based attacks:
  // zsh/mapfile (invisible file I/O via array assignment),
  // zsh/system (sysopen/syswrite two-step file access),
  // zsh/zpty (pseudo-terminal command execution),
  // zsh/net/tcp (network exfiltration via ztcp),
  // zsh/files (builtin rm/mv/ln/chmod that bypass binary checks)
  'zmodload',
  'emulate',  // emulate -c is eval-equivalent
  // 模块 builtin (需要 zmodload 后才能用，但当 defense-in-depth)
  'sysopen', 'sysread', 'syswrite', 'sysseek',
  'zpty',
  'ztcp', 'zsocket',
  'mapfile',
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', 'zf_chown', 'zf_mkdir', 'zf_rmdir', 'zf_chgrp',
])
```

zsh 比 bash 多了一整套**模块加载机制**：`zmodload zsh/system` 之后能用 `sysopen` 直接打开文件（绕过文件 IO 二进制如 cat/cp 的检查）。整个 module 系统几乎都是"隐形"的——传统的"看命令名"完全防不住。

注释里把每个模块的危害说清楚：
- `zsh/system` → `sysopen`/`syswrite` 两步式文件访问
- `zsh/zpty` → 在伪终端里跑命令（绕过 process 监控）
- `zsh/net/tcp` → `ztcp` 直接 TCP 通信（数据外泄）
- `zsh/files` → `zf_rm` 等 builtin（绕过 binary 检查）
- `zsh/mapfile` → 用数组赋值做不可见文件 IO

**所有 zsh 模块化 builtin 都在 deny 名单**——即使主 shell 是 bash，攻击者也可能尝试 `zsh -c '...'` 调用。这是 defense-in-depth 把 zsh 完全标记成不可信。

`'emulate'` 也在名单——`emulate -c 'arbitrary code'` 是个 eval-equivalent。

注意有些 zsh builtin 严格说需要先 `zmodload`，但这里**也加进 deny 列表当 defense-in-depth**——以防：
- module 已经被预加载；
- zmodload 被某种方式 bypass；
- 未来某 zsh 版本默认启用。

## 3.7 COMMAND_SUBSTITUTION_PATTERNS —— 12 种命令替换语法

`bashSecurity.ts:16-41`：

```ts
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /=\(/, message: 'Zsh process substitution =()' },
  {
    pattern: /(?:^|[\s;&|])=[a-zA-Z_]/,
    message: 'Zsh equals expansion (=cmd)',
  },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  { pattern: /\$\[/, message: '$[] legacy arithmetic expansion' },
  { pattern: /~\[/, message: 'Zsh-style parameter expansion' },
  { pattern: /\(e:/, message: 'Zsh-style glob qualifiers' },
  { pattern: /\(\+/, message: 'Zsh glob qualifier with command execution' },
  {
    pattern: /\}\s*always\s*\{/,
    message: 'Zsh always block (try/always construct)',
  },
  { pattern: /<#/, message: 'PowerShell comment syntax' },
]
```

注意几个细节：

### Zsh =cmd expansion

```
=cmd args
```

zsh 里 `=cmd` 等价于 `$(which cmd)`——展开成完整路径。攻击：

```bash
=curl evil.com
```

如果 deny 规则是 `Bash(curl:*)`——它匹配字面 `curl`。但 `=curl` 展开后 zsh 执行 `/usr/bin/curl evil.com`，但解析器看到的 base command 是 `=curl` 不是 `curl` → 绕过 deny。

regex `/(?:^|[\s;&|])=[a-zA-Z_]/` 捕获**词首位置**的 `=` + 字母（区别于 `VAR=val`，那种 `=` 前面跟着字母数字）。

### 最后一条：PowerShell `<#`

```ts
{ pattern: /<#/, message: 'PowerShell comment syntax' },
```

PowerShell 的注释语法。注释（`bashSecurity.ts:38-40`）：

> Defense in depth: Block PowerShell comment syntax even though we don't execute in PowerShell. Added as protection against future changes that might introduce PowerShell execution.

**未来 proofing**——bash 路径上理论上不会执行 PowerShell，但万一未来某个改动引入跨 shell 执行，这条规则提前堵了。这是个有远见的设计。

## 3.8 validateObfuscatedFlags —— 标志混淆

`bashSecurity.ts:1130+` 是 ~400 行的庞然大物，专门处理**用引号/转义混淆危险标志**的攻击。

核心场景：危险标志（如 `rm` 的 `--no-preserve-root`、`curl` 的 `--upload-file`）在 deny 规则里被列。攻击者用引号或转义绕过：

```bash
rm --"no"-preserve-root /
rm $'--no-preserve-root' /
rm $'\055\055no-preserve-root' /
rm '--n''o-preserve-root' /
```

bash 的解析会 unquote 这些都得到 `--no-preserve-root`，但简单的 regex `/--no-preserve-root/` 看不出来。

`validateObfuscatedFlags` 做几件事：
1. **echo 短路**：纯 echo 命令（无 `|` / `&` / `;`）默认是安全的——echo 不接危险标志；
2. **ANSI-C quoting 检测**：识别 `$'...'` 这类 C 风格转义；
3. **多片段拼接检测**：识别 `--"foo"-bar` 或 `--'foo'bar` 这类拼接；
4. **base64/hex 转义检测**；
5. **trailing 转义检测**。

由于规模太大不展开细节，但思路很统一：**用 unquoted form 做比对**，让混淆的标志失去藏身处。

## 3.9 ValidationContext 的多种 unquoted 形式

注意 `ValidationContext` 有 4 种 unquote 形式：

```ts
type ValidationContext = {
  originalCommand: string         // 原始字符串
  baseCommand: string             // argv[0]
  unquotedContent: string         // 仅剥单引号
  fullyUnquotedContent: string    // 剥所有引号 + 处理 redirections
  fullyUnquotedPreStrip: string   // 剥引号但不处理 redirections
  unquotedKeepQuoteChars: string  // 剥引号内容但保留 ' 和 " 符号
  treeSitter?: TreeSitterAnalysis | null
}
```

为什么需要 5 种形式？因为**不同攻击隐藏在不同 unquote 层次里**：

- **`originalCommand`**：检测 `$IFS`、`/proc/environ` 这种字面值；
- **`unquotedContent`**：检测被双引号包裹的元字符；
- **`fullyUnquotedContent`**：检测危险标志（混淆的 `--rm-rf` 等）；
- **`fullyUnquotedPreStrip`**：用于 brace expansion 检测，避免 redirection stripping 制造假阴性；
- **`unquotedKeepQuoteChars`**：用于检测**引号紧邻 `#`** 这种边界情况（如 `'x'#comment`）。

每个 validator **挑合适的 form 用**。这种"一次 extract、多个视角"的设计避免每个 validator 重复做字符串处理。

## 3.10 PermissionResult 的传递语义

每个 validator 返回 `PermissionResult`，下游决策按以下顺序聚合：

```
forEach validator in chain:
  result = validator(ctx)
  switch result.behavior:
    case 'deny':         → 中止链, 返回 deny
    case 'ask':          → 中止链, 返回 ask（携带 message）
    case 'passthrough':  → 继续下一 validator
    case 'allow':        → 通常用于复合判定（少见）
end

→ 全部 passthrough = 默认 allow（实际还要走 bashPermissions 决策）
```

注意：bashSecurity 的检查**不直接 allow**——它最多做到"没检测出问题，passthrough"。最终是否 allow 由 bashPermissions（[04](./04-permission决策核心.md)）决定。

## 3.11 几个隐性设计判断

### 1. 23 个独立 validator 而不是一个大函数

每个 validator 单一职责、独立可测。这看起来啰嗦，但有几个实际好处：
- **加新检查只增不改**：新攻击模式发现 → 加一个新 validator + 一个 checkId；
- **每条规则独立 logEvent**：能 A/B 评估"开这条规则会拒掉多少正常用法"；
- **诚实可见**：23 条规则就是 23 条规则，没有藏在一个 god function 里的隐式判定。

### 2. checkId 是 numeric 而不是字符串

把规则名硬编码进 analytics 是糟糕的——会泄漏 schema、占用空间。numeric ID 让代码层和 analytics 层解耦。

### 3. defense-in-depth 多次出现

`validateProcEnvironAccess` 注释明确说"path validation typically blocks /proc access, but this provides defense-in-depth"。`ZSH_DANGEROUS_COMMANDS` 把"理论上需要 zmodload"的命令也加进 deny。`<#` PowerShell 注释语法在 bash 路径上也防。

每一层都假设上一层可能漏过——这和 [prompt-injection 专题](../prompt-injection/) 的 6 层纵深防御思路完全一致。

### 4. HackerOne reference 写进代码注释

`validateMalformedTokenInjection` 注释明确"This check catches the eval bypass discovered in HackerOne review"。和 [prompt-injection 03 Unicode 清洗](../prompt-injection/03-Unicode清洗管线.md) 的 HackerOne #3086545 引用一样。

**注释里写下漏洞来源**让未来的工程师/LLM agent 能查到完整背景。

### 5. 用 zsh 防 bash 攻击面

zsh 的危险模块清单出现在 bashSecurity 里——即使主 shell 是 bash，攻击者也可能 `zsh -c '...'`。把 zsh 的攻击面预先 deny 是把整个相邻 shell 系统的攻击面都纳入考虑。

### 6. 短路链 + passthrough 默认 = 安全设计

链路上任一 validator ask/deny → 中止；全部 passthrough → 默认放过。这是个**保守的** "**fail-open 但仅在所有 validator 都 explicit OK 后**" 的设计——避免新增 validator 默认导致大规模 false positive。

新增检查时只需 return `ask`/`passthrough`/`deny`，不会破坏其它 validator 的语义。

## 3.12 23 个 validator 总览表

| checkId | validator | 防御什么 | 示例攻击 |
|---|---|---|---|
| 1 | `validateIncompleteCommands` | 不完整/截断命令 | `rm -r"` (引号未闭合) |
| 2 | `validateJqCommand` (system) | jq 用 system function 执行命令 | `jq 'system("rm -rf /")'` |
| 3 | `validateJqCommand` (file args) | jq 读敏感文件 | `jq -s '.' < /etc/passwd` |
| 4 | `validateObfuscatedFlags` | 混淆的危险 flag | `rm $'--no-preserve-root' /` |
| 5 | `validateShellMetacharacters` | 危险 shell 元字符 | `cmd1 ; cmd2` 跨界 |
| 6 | `validateDangerousVariables` | 危险变量赋值 | `LD_PRELOAD=...` |
| 7 | `validateNewlines` | 命令中含换行 | `echo \nrm -rf /` |
| 8 | `validateDangerousPatterns` (cmdsub) | 命令替换 | `cmd $(evil)` |
| 9 | `validateDangerousPatterns` (input redir) | 输入重定向到危险源 | `cmd < /etc/shadow` |
| 10 | `validateDangerousPatterns` (output redir) | 输出重定向到危险目标 | `cmd > /etc/passwd` |
| 11 | `validateIFSInjection` | IFS 词分隔注入 | `cat$IFS/etc/passwd` |
| 12 | `validateGitCommit` | git commit 中含命令替换 | `git commit -m "$(rm -rf /)"` |
| 13 | `validateProcEnvironAccess` | /proc 读环境变量 | `cat /proc/self/environ` |
| 14 | `validateMalformedTokenInjection` | malformed token + 分隔符 | `echo {"hi":"hi;evil"}` |
| 15 | `validateBackslashEscapedWhitespace` | backslash 转义空白 | `rm\<tab>-rf` |
| 16 | `validateBraceExpansion` | brace 扩展危险 | `rm {a,b,c,/}` |
| 17 | `validateControlCharacters` | 控制字符 | `\x1b[2K rm` |
| 18 | `validateUnicodeWhitespace` | Unicode 空白 | NBSP / em space |
| 19 | `validateMidWordHash` | 单词内 `#` | `'x'#comment` |
| 20 | `validateZshDangerousCommands` | zsh 危险命令 | `zmodload zsh/system` |
| 21 | `validateBackslashEscapedOperators` | backslash 转义操作符 | `cmd1\;cmd2` |
| 22 | `validateCommentQuoteDesync` | 注释/引号失同步 | `echo "x"#'rest` |
| 23 | `validateQuotedNewline` | 引号包裹的换行 | `cmd "a\nb"` |

每条规则都来自一个真实的攻击场景。

## 3.13 小结

- `bashSecurity.ts` 2592 行 = 23 个独立 validator + 7 类辅助函数；
- 每个 validator 单一职责，链路上短路 ask/deny、累积 passthrough；
- 多种 unquote 形式（5 种）让不同 validator 选合适的视角；
- numeric checkId 服务于 analytics（避免泄漏 schema）；
- `$IFS` / `/proc/environ` / malformed token 这类经典攻击都有专门 validator；
- zsh 模块系统（zmodload/sysopen/zpty 等）整体被列入 deny，即使主 shell 是 bash；
- PowerShell `<#` 注释也加进 deny 是有远见的 future-proofing；
- defense-in-depth 多次出现，每层独立，呼应 [prompt-injection 专题](../prompt-injection/) 的纵深防御思路。

下一篇 → [04 permission 决策核心](./04-permission决策核心.md)
