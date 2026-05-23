# 06 sed 与 wrapper specs

> 两个看起来不相关的话题，其实都在解决同一问题：**让"长得不像直接命令"的形式也能被精确决策**。本篇看 sed 模拟执行的设计、7 个 wrapper spec 的注册机制。

## 6.1 sed -i 的问题

`sed -i 's/foo/bar/g' file.txt` 看起来是 read 命令——它是文本处理工具。但 `-i` 标志让它**直接覆盖文件**，没有 diff、没有备份、不可逆。

如果按"sed 是 read-only"判定 → 允许 → 模型能用 sed 修改任何文件。如果按"sed 不安全" → 拒绝所有 sed → 真正的 read 用法（`sed -n '1,10p' file`）也被拦。

Claude Code 的解法不是"判断 yes/no"，而是 **"把 `sed -i` 重定向到 FileEdit 工具"** —— 解析 sed 表达式、模拟成等价的文件编辑、在 FileEdit 的常规 diff UI 里走审查流程。

## 6.2 SedEditInfo —— 解析结果结构

`sedEditParser.ts:25-36`：

```ts
export type SedEditInfo = {
  /** The file path being edited */
  filePath: string
  /** The search pattern (regex) */
  pattern: string
  /** The replacement string */
  replacement: string
  /** Substitution flags (g, i, etc.) */
  flags: string
  /** Whether to use extended regex (-E or -r flag) */
  extendedRegex: boolean
}
```

`parseSedEditCommand(command)` 返回 `SedEditInfo | null`：

- 成功 → 上游知道这是个 sed -i edit，走模拟路径；
- 失败 → 上游按普通 bash 命令处理（要么走 sedValidation 看是不是 read-only 模式，要么走 ask）。

## 6.3 parseSedEditCommand 的限制

`sedEditParser.ts:48+`：

```ts
export function parseSedEditCommand(command: string): SedEditInfo | null {
  const trimmed = command.trim()

  // Must start with sed
  const sedMatch = trimmed.match(/^\s*sed\s+/)
  if (!sedMatch) return null

  const withoutSed = trimmed.slice(sedMatch[0].length)
  const parseResult = tryParseShellCommand(withoutSed)
  if (!parseResult.success) return null
  const tokens = parseResult.tokens

  // Extract string tokens only
  const args: string[] = []
  for (const token of tokens) {
    if (typeof token === 'string') {
      args.push(token)
    } else if (
      typeof token === 'object' &&
      token !== null &&
      'op' in token &&
      token.op === 'glob'
    ) {
      // Glob patterns are too complex for this simple parser
      return null
    }
  }
  // ... 后续解析 -i / -E / 's///' / filename
}
```

故意只支持**最简单**的形式：

- 必须 `sed` 开头（不允许 `cat foo | sed ...` 或 `nohup sed`）；
- 不能有 glob（`*.txt`）；
- 只解析单一 substitution 表达式（不支持 `sed -e ... -e ...` 多个 -e）；
- 不接受复合命令（必须是单纯的 sed 调用）。

**故意保守**——能精确模拟的才走模拟路径，不能的走 ask。这种"narrow path of acceptance" 设计避免误处理产生 bug。

注释里特别处理了 macOS sed 的兼容：

```ts
// On macOS, -i requires a suffix argument (even if empty string)
// Check if next arg looks like a backup suffix (empty, or starts with dot)
```

macOS BSD sed 的 `-i` 要求紧跟一个 backup suffix（即使空字符串），GNU sed 不要求。这种平台差异要在解析时处理。

## 6.4 BRE → ERE 转换

`sedEditParser.ts:9-21`：

```ts
const BACKSLASH_PLACEHOLDER = '\x00BACKSLASH\x00'
const PLUS_PLACEHOLDER = '\x00PLUS\x00'
const QUESTION_PLACEHOLDER = '\x00QUESTION\x00'
const PIPE_PLACEHOLDER = '\x00PIPE\x00'
const LPAREN_PLACEHOLDER = '\x00LPAREN\x00'
const RPAREN_PLACEHOLDER = '\x00RPAREN\x00'
```

sed 默认用 **BRE（Basic Regular Expression）**：`+` / `?` / `|` / `(` / `)` 都要 backslash 前缀（`\+` `\?` `\|` `\(` `\)`）才有特殊含义。

`-E` / `-r` 启用 **ERE（Extended Regular Expression）**：上面那些符号反过来——不带 backslash 才有特殊含义。

Claude Code 的 FileEdit 内部用 JS regex（ERE 风格）。所以从 BRE pattern 转 ERE 时需要：

- `\+` → `+`（去 backslash）
- `+` → `\+`（加 backslash）

用 null-byte 占位符做两步转换（避免互相干扰）。这是经典的 placeholder trick——和 [02 命令解析管线](./02-命令解析管线.md) 的 `CMDSUB_PLACEHOLDER` 同样思路。

## 6.5 BashTool.checkPermissions 与 sed 的连接

回顾 [04 permission 决策核心](./04-permission决策核心.md) 的 8 步决策树，第 5b 步是 sed：

```ts
// 5b. Check sed constraints (blocks dangerous sed operations before mode auto-allow)
const sedConstraintResult = checkSedConstraints(input, toolPermissionContext)
if (sedConstraintResult.behavior !== 'passthrough') {
  return sedConstraintResult
}
```

`checkSedConstraints` 走 `sedValidation.ts` 判定 sed 命令是否安全模式（只读模式如 `sed -n '1p'`、line printing 等）。

如果不是只读模式（即真的要改文件），且不是 `_simulatedSedEdit` 路径，会**触发 ask**。用户点确认时，permission dialog 调 `parseSedEditCommand` 看能否解析成可模拟形式：

```
能解析:
  → 弹 FileEdit 风格的 diff preview
  → 用户审查通过
  → permission dialog 把 SedEditInfo 转成 { filePath, newContent }
  → 塞进 input._simulatedSedEdit
  → 走 BashTool.call()

  call() 看到 _simulatedSedEdit:
    → 短路: applySedEdit() 直接写文件
    → 不 spawn sed 进程
    → 不走 sandbox
    → fileHistoryTrackEdit() 让 /undo 可恢复

不能解析:
  → 弹普通 Bash command 确认
  → 用户审查文本命令
  → 真的 spawn sed
```

这就是 [01 工具入口](./01-工具入口与prompt定义.md) 提到的 `_simulatedSedEdit` 字段的设计动机。

## 6.6 isSedInPlaceEdit —— 渲染时的辅助

BashTool.tsx 用 `parseSedEditCommand` 在 UI 显示时做小优化：

```ts
userFacingName(input) {
  if (input?.command) {
    const sedInfo = parseSedEditCommand(input.command)
    if (sedInfo) {
      return fileEditUserFacingName({ file_path: sedInfo.filePath, old_string: 'x' })
    }
  }
  // ...
}
```

如果 input 是 sed in-place edit，UI 显示**FileEdit 的样式**而不是 BashTool 的样式——给用户的视觉提示是"这是个文件编辑操作"。

这是个**统一用户体验**的设计——不让用户因为模型选了 sed 而看到不同的 UI。

## 6.7 sedValidation.ts —— 只读 sed 模式识别

`sedValidation.ts` 684 行的核心问题：**哪些 sed 命令真的是只读？**

```bash
sed -n '1p' file.txt           # 只打印第一行 - 只读
sed -n '/foo/p' file.txt       # grep 风格 - 只读
sed 's/foo/bar/' file.txt      # 替换并打印到 stdout - 只读（没 -i）
sed -i 's/foo/bar/' file.txt   # in-place - 写
sed -i.bak '...' file.txt      # in-place 带备份 - 写
sed -n '1,10p; 20p' file.txt   # 多个 print 命令 - 只读
sed -e 's/a/b/' -e 's/c/d/' file.txt  # 多 -e - 写（默认不 -i 也输出到 stdout，但模型行为难判定）
```

`sedValidation` 用模式匹配识别已知的"只读模式"：

```ts
// isLinePrintingCommand: -n 'NUMBER p' 形式
// isSimpleSubstitution: 's/a/b/' 但没 -i
// isAllowedScriptForm: 已知良好的 -e 用法
// ...
```

每个模式有自己的 flag allowlist + 表达式语法验证。

`sedCommandIsAllowedByAllowlist` 是顶层入口——如果命令匹配任何一种允许模式 → 返回 true → 允许。不匹配 → 走 ask。

### validateFlagsAgainstAllowlist —— 处理组合标志

```ts
function validateFlagsAgainstAllowlist(flags: string[], allowedFlags: string[]): boolean {
  for (const flag of flags) {
    if (flag.startsWith('-') && !flag.startsWith('--') && flag.length > 2) {
      // 组合 flag 如 -nE 或 -Er
      for (let i = 1; i < flag.length; i++) {
        const singleFlag = '-' + flag[i]
        if (!allowedFlags.includes(singleFlag)) return false
      }
    } else {
      if (!allowedFlags.includes(flag)) return false
    }
  }
  return true
}
```

处理 unix 短 flag 的**组合形式**：`-nE` = `-n -E`，需要拆开逐个检查。

## 6.8 wrapper specs 系统 —— 另一套透视机制

`utils/bash/specs/` 下 7 个文件 + index + registry，注册了"wrapper 命令"的语义：

```
specs/
├── index.ts          # 7 个 spec 的注册数组
├── alias.ts
├── nohup.ts
├── pyright.ts
├── sleep.ts
├── srun.ts
├── time.ts
└── timeout.ts
```

### CommandSpec 类型

`registry.ts:5-30`：

```ts
export type CommandSpec = {
  name: string
  description?: string
  subcommands?: CommandSpec[]
  args?: Argument | Argument[]
  options?: Option[]
}

export type Argument = {
  name?: string
  description?: string
  isDangerous?: boolean
  isVariadic?: boolean
  isOptional?: boolean
  isCommand?: boolean   // ← wrapper 命令标志: e.g. timeout, sudo
  isModule?: string | boolean   // python -m
  isScript?: boolean   // node script.js
}
```

`isCommand: true` 是关键——告诉系统"这个参数本身是另一个命令"。

### 例：timeout spec

```ts
const timeout: CommandSpec = {
  name: 'timeout',
  description: 'Run a command with a time limit',
  args: [
    {
      name: 'duration',
      description: 'Duration to wait before timing out (e.g., 10, 5s, 2m)',
      isOptional: false,
    },
    {
      name: 'command',
      description: 'Command to run',
      isCommand: true,  // ← 关键
    },
  ],
}
```

`timeout` 接两个位置参数：duration（数字 / 时长字符串）+ command（**自身是一个命令**）。`isCommand` 让 `getCommandPrefixStatic` 知道要**继续解析这个参数**。

### 例：nohup spec

```ts
const nohup: CommandSpec = {
  name: 'nohup',
  description: 'Run a command immune to hangups',
  args: {
    name: 'command',
    description: 'Command to run with nohup',
    isCommand: true,
  },
}
```

更简单——只有一个 `command` 参数。

### getCommandPrefixStatic —— 透视入口

`prefix.ts:34+`：

```ts
export async function getCommandPrefixStatic(
  command: string,
  recursionDepth = 0,
  wrapperCount = 0,
): Promise<{ commandPrefix: string | null } | null> {
  if (wrapperCount > 2 || recursionDepth > 10) return null
  // ...
  const spec = await getCommandSpec(cmd)
  let isWrapper =
    WRAPPER_COMMANDS.has(cmd) ||
    (spec?.args && toArray(spec.args).some(arg => arg?.isCommand))

  // 特殊：subcommand 优先
  if (isWrapper && args[0] && isKnownSubcommand(args[0], spec)) {
    isWrapper = false
  }

  const prefix = isWrapper
    ? await handleWrapper(cmd, args, recursionDepth, wrapperCount)
    : await buildPrefix(cmd, args, spec)
  // ...
}
```

是 wrapper（spec 含 `isCommand: true` 的 arg）→ 递归处理内层命令；否则按普通命令构建 prefix。

**限制递归深度**：`wrapperCount > 2 || recursionDepth > 10` → 拒绝。防止恶意构造的深度嵌套 `timeout timeout timeout timeout ...`。

### subcommand vs wrapper 的歧义处理

`prefix.ts:51-54`：

```ts
// Special case: if the command has subcommands and the first arg matches a subcommand,
// treat it as a regular command, not a wrapper
if (isWrapper && args[0] && isKnownSubcommand(args[0], spec)) {
  isWrapper = false
}
```

注释（解释为什么需要这条）：

> git spec has isCommand args for aliases

git 的 spec 标了某些 args 为 `isCommand`（如 `git <alias>` 中的 alias 可能展开成任意命令）。但 `git status` 这种第一个参数是 **subcommand 而不是命令**——不应该把 `status` 当作"另一个命令"递归解析。

修复：如果 `args[0]` 是已知的 subcommand，按普通命令处理。

## 6.9 两套 wrapper 透视并存

注意现在有**两套** wrapper 透视：

1. **`stripSafeWrappers`**（[04 permission 决策](./04-permission决策核心.md)，`bashPermissions.ts:524`）：字符串级，硬编码 5 个 wrapper（timeout/time/nice/stdbuf/nohup）；
2. **`getCommandPrefixStatic`**（本篇，`prefix.ts:34`）：spec 驱动，7 个 spec（+ fig 自动补全）。

为什么有两套？

- `stripSafeWrappers` 是 **regex-based**，快、可靠、用于 permission decision 的高频路径；
- `getCommandPrefixStatic` 是 **spec-based**，可扩展、能从 `@withfig/autocomplete` 加载几百个 spec、用于 UI 显示 / 类型推断。

两套的**精确语义略有差异**——`stripSafeWrappers` 支持 nice / stdbuf（实用），`getCommandPrefixStatic` 走 spec 系统（更通用）。两套都基于"`isCommand` 透视"的思想，但实现独立。

这也是 [05 path/readonly](./05-只读vs可写与路径校验.md) 提到的 "3 处必须同步" 中的两处——加新 wrapper 要同时改 regex 和 spec。

## 6.10 sleep spec 的特殊性

`specs/sleep.ts`（应该很短）的 spec 让 sleep 不被当作普通命令——它会被 `detectBlockedSleepPattern` 截获（[01 工具入口](./01-工具入口与prompt定义.md)）：

```ts
// BashTool.tsx:322-337
export function detectBlockedSleepPattern(command: string): string | null {
  const parts = splitCommand_DEPRECATED(command)
  if (parts.length === 0) return null
  const first = parts[0]?.trim() ?? ''
  const m = /^sleep\s+(\d+)\s*$/.exec(first)
  if (!m) return null
  const secs = parseInt(m[1]!, 10)
  if (secs < 2) return null  // sub-2s sleeps OK

  // `sleep N` alone → "what are you waiting for?"
  // `sleep N && check` → "use Monitor { command: check }"
  const rest = parts.slice(1).join(' ').trim()
  return rest ? `sleep ${secs} followed by: ${rest}` : `standalone sleep ${secs}`
}
```

`sleep N (N≥2)` 作为首个命令 → 返回错误信息 → BashTool.validateInput 拒绝执行。

为什么？因为模型有**滥用 sleep 的 anti-pattern**——`sleep 5 && check_status`（用 sleep 模拟 polling）。这浪费 wall-clock time、不灵活。Claude Code 推动模型用 **Monitor 工具** 流式监听事件。

`prompt.ts:310-328` 在 prompt 里也有专门一节教模型不要用 sleep。

## 6.11 几个隐性设计判断

### 1. 把 sed -i 重定向到 FileEdit

不是 deny sed-i，而是**用更安全的工具替代实现**。这是个非常优雅的设计模式：

- 保留功能（用户能"修改文件"）；
- 获得审计能力（diff preview / undo）；
- 不打破模型的工具使用直觉（它仍然写 `sed -i ...`，只是结果走 FileEdit 路径）。

类似的设计还可以应用到 awk、find -exec 等"看起来读但实际能写"的命令——目前 Claude Code 只对 sed 做这套，但思路是 generalizable。

### 2. 故意 narrow 的 sed 解析

`parseSedEditCommand` 只支持最简单的形式——能精确模拟才走模拟。复杂形式回退到普通 ask。

这种"narrow path of acceptance" 比"broad coverage with edge case bugs" 安全得多。每加一种新形式要重新评估 edge cases，所以保持小。

### 3. spec 系统 vs regex 系统并存

两套 wrapper 透视各有用途：
- regex 系统快、用于热路径；
- spec 系统可扩展、用于补全和 UI。

并存是工程现实——重构成单一系统需要重新跑全部测试 + 性能对比。

### 4. fig 自动补全 spec 作为 fallback

```ts
const module = await import(`@withfig/autocomplete/build/${command}.js`)
```

`@withfig/autocomplete` 是个开源项目，提供几百个 CLI 工具的 spec。Claude Code 用它做 fallback：

- 内部 7 个 spec 优先；
- 找不到 → 尝试加载 fig spec；
- 都找不到 → 当普通命令。

这是个**杠杆开源生态**的设计——不重写 git/docker/kubectl 的 spec，而是借用现成的。

### 5. 平台差异 baked into 解析

`-i` 的 macOS vs GNU 处理（macOS 强制 backup suffix）写在 parser 里。这种"为多平台正确解析"的细节是工业代码的累积——每条都来自一次 bug report。

### 6. sleep 不是被 deny 而是被引导

`detectBlockedSleepPattern` 不直接拒绝 `sleep`——它返回**带建议的错误消息**（"use Monitor tool"）。模型看到错误会读建议、改用 Monitor。

这是"教育性错误信息"——比单纯 deny 更利于模型学习正确用法。和 [01 工具入口](./01-工具入口与prompt定义.md) 提到的"教育模式" 一脉相承。

### 7. wrapper 深度限制防 ReDoS

`wrapperCount > 2 || recursionDepth > 10` 限制递归深度。攻击者构造 `timeout timeout timeout ... ` 试图引发指数复杂度 → 早期拒绝。

这种"在递归入口设硬上限"是处理用户/模型输入的标配。

## 6.12 小结

- sed -i 的解法不是 deny，是**重定向到 FileEdit 工具**——保留功能 + 获得审计；
- `parseSedEditCommand` 故意只支持最简单形式，narrow path of acceptance；
- BRE → ERE 转换用 null-byte placeholder 避免互相干扰；
- `_simulatedSedEdit` 字段是 permission dialog 和 BashTool.call() 之间的私有信道；
- sedValidation 识别只读 sed 模式（line printing 等），允许它们走 read-only auto-allow；
- wrapper specs 系统（7 个 spec + fig autocomplete）是另一套透视机制，spec-based 可扩展；
- 两套 wrapper 透视（regex 和 spec）并存，各有用途，必须同步；
- sleep 不被 deny 而被引导到 Monitor 工具；
- 平台差异（macOS sed -i 强制 backup suffix）写进解析器。

下一篇 → [07 Sandbox 沙箱机制](./07-Sandbox沙箱机制.md)
