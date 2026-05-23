# 01 工具入口与 prompt 定义

> BashTool 暴露给模型的"表面"——它的工具定义（schema + description）、模型看到的指令、`call()` 主流程。读完本篇会知道：模型怎么调 Bash、Bash 怎么决策、结果怎么回到模型。

## 1.1 BashTool 的工具定义结构

`tools/BashTool/BashTool.tsx:420-825` 是一个标准的 `buildTool` 调用：

```ts
export const BashTool = buildTool({
  name: BASH_TOOL_NAME,                    // 'Bash'
  searchHint: 'execute shell commands',
  maxResultSizeChars: 30_000,              // 30K chars 后转 persisted output
  strict: true,

  async description({ description }) {
    return description || 'Run shell command'
  },
  async prompt() {
    return getSimplePrompt()                // 工具的完整说明文本
  },

  isConcurrencySafe(input) {                // 只读命令可并行
    return this.isReadOnly?.(input) ?? false
  },
  isReadOnly(input) {                       // 走 readOnlyValidation 判定
    const compoundCommandHasCd = commandHasAnyCd(input.command)
    const result = checkReadOnlyConstraints(input, compoundCommandHasCd)
    return result.behavior === 'allow'
  },

  toAutoClassifierInput(input) {            // 给 auto-mode classifier 的输入
    return input.command
  },

  async preparePermissionMatcher({ command }) { ... },  // 见 1.6
  isSearchOrReadCommand(input) { ... },

  get inputSchema() { return inputSchema() },
  get outputSchema() { return outputSchema() },

  userFacingName(input) { ... },
  getToolUseSummary(input) { ... },
  getActivityDescription(input) { ... },

  async validateInput(input) { ... },       // sleep 检测等
  async checkPermissions(input, context) {  // 权限决策入口
    return bashToolHasPermission(input, context)
  },

  // 渲染相关
  renderToolUseMessage, renderToolUseProgressMessage,
  renderToolUseQueuedMessage, renderToolResultMessage,
  renderToolUseErrorMessage,

  extractSearchText({ stdout, stderr }) { ... },
  mapToolResultToToolResultBlockParam(...) { ... },  // 结果转 tool_result block

  async call(input, toolUseContext, _canUseTool, parentMessage, onProgress) {
    // 主流程：runShellCommand 异步生成器 → 累积输出 → 处理结果
  },

  isResultTruncated(output) { ... },
})
```

每个回调对应一个明确的职责。最关键的 4 个：

| 回调 | 职责 | 见下文 |
|------|------|-------|
| `prompt()` | 返回工具的描述文本（拼进 API `tools[].description`） | 1.2 |
| `inputSchema` | 输入参数的 Zod schema | 1.3 |
| `checkPermissions()` | 决定能不能执行（allow / deny / ask） | 1.5 |
| `call()` | 真正的执行入口 | 1.7 |

## 1.2 prompt 文本的拼接结构

`tools/BashTool/prompt.ts:275-369` 的 `getSimplePrompt()` 是工具描述的组装函数。它返回的字符串会被拼进 API 请求的 `tools[].description` 字段——模型在 system prompt 之外看到的"这个工具能做什么、该怎么用"全部来自这里。

结构（按出现顺序）：

```
1. 基础说明
   "Executes a given bash command and returns its output."
   "The working directory persists between commands, but shell state does not."

2. ⚠️ 优先使用专用工具的强提醒
   "IMPORTANT: Avoid using this tool to run cat, head, tail, sed, awk, or echo..."
   - File search: Use Glob (NOT find or ls)
   - Content search: Use Grep (NOT grep or rg)
   - Read files: Use Read (NOT cat/head/tail)
   - Edit files: Use Edit (NOT sed/awk)
   - Write files: Use Write (NOT echo >/cat <<EOF)
   - Communication: Output text directly (NOT echo/printf)

3. # Instructions
   - 创建文件前先 ls 验证父目录
   - 含空格的路径要双引号
   - 尽量用绝对路径不要 cd
   - timeout 上限（默认 / 最大）
   - run_in_background 用法
   - 复合命令的最佳实践
     - 并行用多个 Bash 调用
     - 串行用 &&
     - ; 只在不关心失败时用
     - 不要用换行分割命令
   - git 命令
     - 优先新 commit, 不要 amend
     - 破坏性操作前评估
     - 不要 --no-verify
   - 避免 sleep
     - 不要在能立即跑的命令之间 sleep
     - 长命令用 run_in_background
     - sleep N (N≥2) 被禁止（MONITOR_TOOL gate 下）
     - 用 Monitor 工具流式监听

4. # Command sandbox（如果沙箱启用）
   - 沙箱限制说明
   - filesystem.read.denyOnly / allowWithinDeny
   - filesystem.write.allowOnly / denyWithinAllow
   - network.allowedHosts / deniedHosts
   - 允许的 unix sockets
   - TMPDIR 优先 /tmp
   - dangerouslyDisableSandbox 使用条件

5. # Committing changes with git（如果未禁用 git 指令）
   - Ant 用户：指向 /commit 和 /commit-push-pr skill
   - 外部用户：完整 7 步 commit 流程 + heredoc 示例
   - Git Safety Protocol 7 条
   - 创建 PR 的完整流程
   - 命令注释要求

6. # Other common operations
   - gh api 用法
```

注意几个设计点：

### 1. "优先用专用工具" 放在最显眼位置

第 2 节是个**强提醒**——告诉模型不要用 BashTool 跑 `cat/grep/find/sed/echo` 这些，应该用对应的专用工具（Read/Grep/Glob/Edit/Write）。

这不是性能优化，是**护栏**：
- 专用工具有自己的护栏（FileRead 加行号前缀 + malware reminder，见 [prompt-injection 02](../prompt-injection/02-FileRead双护栏.md)）；
- 走 BashTool 会绕过这些护栏；
- 所以系统提示先告诉模型"能用专用工具就别用 Bash"。

### 2. sandbox 配置直接拼进 prompt

`getSimpleSandboxSection()`（`prompt.ts:172-273`）会把当前的沙箱配置序列化进 prompt——让模型知道**哪些目录可读、哪些可写、哪些网络可访问**。

关键设计：

```ts
// 把 /private/tmp/claude-1001/ 换成 $TMPDIR 让 cache 跨用户共享
const claudeTempDir = getClaudeTempDir()
const normalizeAllowOnly = (paths: string[]): string[] =>
  [...new Set(paths)].map(p => (p === claudeTempDir ? '$TMPDIR' : p))
```

注释（`prompt.ts:185-187`）：

> Replace the per-UID temp dir literal (e.g. /private/tmp/claude-1001/) with "$TMPDIR" so the prompt is identical across users — avoids busting the cross-user global prompt cache.

为了 prompt cache 命中，把"每用户不同"的字段统一替换成符号——这种细节在 [messages-pipeline 07](../messages-pipeline/07-attachment-normalizer.md) 也出现过（relevant_memories 用存储 header 而非重算）。同样的设计理念：**任何随机/用户特定的字符串都会破坏 cache prefix**。

`dedup()` 函数（`prompt.ts:167-170`）也是为了 cache：去重避免 `~/.cache` 在 `allowOnly` 里重复 3 次（注释说节省 ~150-200 tokens/request）。

### 3. git 指令独立成节、ant 用户走 skill 化路径

第 5 节的 git 指令对**ant 用户**和**外部用户**走两条不同的路径（`prompt.ts:56-160`）：

- **Ant**：精简指令 + 指向 `/commit` 和 `/commit-push-pr` skill；
- **外部**：完整的 7 步 commit 流程 + Git Safety Protocol + PR 创建说明。

这是一个**渐进的简化策略**——内部用户先用 skill 化的方式验证，验证好了再放到外部。

### 4. undercover 模式

`undercoverSection`（`prompt.ts:46-52`）：

```ts
const undercoverSection =
  process.env.USER_TYPE === 'ant' && isUndercover()
    ? getUndercoverInstructions() + '\n'
    : ''
```

"undercover" 是 ant 内部的"假装不是 Claude"模式——让模型在 commit 里不暴露内部 codename。注释（`prompt.ts:50-52`）：

> Defense-in-depth: undercover instructions must survive even if the user has disabled git instructions entirely. Attribution stripping and model-ID hiding are mechanical and work regardless, but the explicit "don't blow your cover" instructions are the last line of defense against the model volunteering an internal codename in a commit message.

注意 "last line of defense" 措辞——这是把 prompt-engineering 当**纵深防御的一部分**。和 [prompt-injection 06 元防御指令](../prompt-injection/06-元防御指令与端到端模拟.md) 是同一类设计哲学。

## 1.3 inputSchema —— 4 个字段

`BashTool.tsx:227-264` 的 `fullInputSchema`：

```ts
const fullInputSchema = lazySchema(() => z.strictObject({
  command: z.string().describe('The command to execute'),
  timeout: semanticNumber(z.number().optional())
    .describe(`Optional timeout in milliseconds (max ${getMaxTimeoutMs()})`),
  description: z.string().optional().describe(`Clear, concise description...`),
  run_in_background: semanticBoolean(z.boolean().optional()).describe(...),
  dangerouslyDisableSandbox: semanticBoolean(z.boolean().optional()).describe(...),
  _simulatedSedEdit: z.object({
    filePath: z.string(),
    newContent: z.string()
  }).optional().describe('Internal: pre-computed sed edit result from preview')
}))
```

4 个面向模型的字段 + 1 个内部字段：

| 字段 | 类型 | 角色 |
|------|------|------|
| `command` | string | 要执行的命令 |
| `timeout` | number? | 超时（ms），默认/最大有上限 |
| `description` | string? | 命令的人话描述（5-10 字，用于 UI） |
| `run_in_background` | boolean? | 后台运行 |
| `dangerouslyDisableSandbox` | boolean? | 跳过 sandbox（要权限） |
| `_simulatedSedEdit` | object? | **内部字段**，永不暴露给模型 |

### 关键细节 1：`_simulatedSedEdit` 永不进 schema

`BashTool.tsx:249-259`：

```ts
// Always omit _simulatedSedEdit from the model-facing schema. It is an internal-only
// field set by SedEditPermissionRequest after the user approves a sed edit preview.
// Exposing it in the schema would let the model bypass permission checks and the
// sandbox by pairing an innocuous command with an arbitrary file write.
const inputSchema = lazySchema(() => isBackgroundTasksDisabled
  ? fullInputSchema().omit({ run_in_background: true, _simulatedSedEdit: true })
  : fullInputSchema().omit({ _simulatedSedEdit: true }))
```

这个字段是 sed 模拟执行机制（[06 sed 与 wrapper specs](./06-sed与wrapper-specs.md)）用的——permission dialog 在用户预览 sed 编辑后，把 pre-computed 结果塞进这个字段直接执行。

注释明确：**暴露给模型 = bypass 整套防御**。所以总是 omit。

### 关键细节 2：`semanticBoolean` 和 `semanticNumber`

模型有时会返回 `"true"` 字符串而不是 boolean，或者 `"30000"` 字符串而不是 number。`semanticBoolean` / `semanticNumber` 在 Zod 层做语义转换，让 `"true"` → `true`、`"30000"` → `30000`。

这是个工程妥协——理想情况模型应该严格 typed，但实际上需要软容错。

### 关键细节 3：`run_in_background` 可被环境关闭

```ts
const isBackgroundTasksDisabled =
  isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_BACKGROUND_TASKS)
```

模块加载时 freeze 了一个 flag。如果禁用了，`run_in_background` 字段直接从 schema 移除——模型看不到这个字段，自然不会去用。

## 1.4 outputSchema —— 12 个字段

`BashTool.tsx:279-294`：

```ts
const outputSchema = lazySchema(() => z.object({
  stdout: z.string(),
  stderr: z.string(),
  rawOutputPath: z.string().optional(),
  interrupted: z.boolean(),
  isImage: z.boolean().optional(),
  backgroundTaskId: z.string().optional(),
  backgroundedByUser: z.boolean().optional(),
  assistantAutoBackgrounded: z.boolean().optional(),
  dangerouslyDisableSandbox: z.boolean().optional(),
  returnCodeInterpretation: z.string().optional(),
  noOutputExpected: z.boolean().optional(),
  structuredContent: z.array(z.any()).optional(),
  persistedOutputPath: z.string().optional(),
  persistedOutputSize: z.number().optional(),
}))
```

`Out` 类型是 BashTool 在内部传递结果用的"信封"。注意几个特殊字段：

- `persistedOutputPath` / `persistedOutputSize`：大于 30KB 的输出写到磁盘，路径放在这里；
- `assistantAutoBackgrounded`：assistant-mode 下命令运行超过 15 秒自动后台化；
- `backgroundedByUser`：用户按 Ctrl+B 手动后台化；
- `returnCodeInterpretation`：非零退出码的语义解释（如 `grep` 退出 1 = "no match"，不是错误）。

最后两个字段是为了**让模型理解 exit code**——`grep` 没匹配到时退出 1 不应该被视为错误，但模型默认会把所有非零 exit 当错误处理。`interpretCommandResult`（在 `commandSemantics.ts`）做这个语义判断。

## 1.5 权限决策 checkPermissions()

`BashTool.tsx:539-541`：

```ts
async checkPermissions(input, context): Promise<PermissionResult> {
  return bashToolHasPermission(input, context)
}
```

只有一行——把整个决策委托给 `bashToolHasPermission()`。

这个函数在 `bashPermissions.ts` 里，是整个权限系统的核心，2621 行代码绝大部分围绕它展开。完整拆解在 [04 permission 决策核心](./04-permission决策核心.md)，这里只看决策返回什么：

```ts
type PermissionResult = {
  behavior: 'allow' | 'deny' | 'ask' | 'passthrough'
  // ...
}
```

4 种结果：

- `allow`：自动放行（如规则匹配、或只读命令在 yolo mode）；
- `deny`：自动拒绝（如命令在 disallow 规则里、或检测到危险模式）；
- `ask`：弹用户确认（默认情况）；
- `passthrough`：交给下游决策（很少用）。

## 1.6 preparePermissionMatcher —— hook 的"如果匹配" 用

`BashTool.tsx:445-468`：

```ts
async preparePermissionMatcher({ command }) {
  const parsed = await parseForSecurity(command)
  if (parsed.kind !== 'simple') {
    return () => true  // 复杂命令: 保守地都触发 hook
  }
  const subcommands = parsed.commands.map(c => c.argv.join(' '))
  return pattern => {
    const prefix = permissionRuleExtractPrefix(pattern)
    return subcommands.some(cmd => {
      if (prefix !== null) {
        return cmd === prefix || cmd.startsWith(`${prefix} `)
      }
      return matchWildcardPattern(pattern, cmd)
    })
  }
}
```

这个函数为 hook 系统的 `if` 字段提供 matcher。注释（`BashTool.tsx:448-450`）：

> Hook `if` filtering is "no match → skip hook" (deny-like semantics), so compound commands must fire the hook if ANY subcommand matches.

含义：用户配置 `PreToolUse` hook 时可以加 `if: 'Bash(git push:*)'` 条件。如果命令是 `ls && git push`，hook 必须触发（因为 git push 在里面）。`preparePermissionMatcher` 做这个"逐 subcommand 匹配"。

注意 `c.argv.join(' ')`——这是去掉了前导环境变量赋值的"纯命令"。`FOO=bar git push` 的 argv 是 `['git', 'push']`，所以 `Bash(git *)` 仍能匹配。这是后面 04 篇要详细讲的"看穿前缀"机制的早期应用。

## 1.7 call() 主流程

`BashTool.tsx:624-820` 是 BashTool 的执行入口。整体流程：

```
0. _simulatedSedEdit 短路
   if (input._simulatedSedEdit) → applySedEdit() 直接走文件写入路径
                                   (不走 spawn, 不走 sandbox)

1. 准备 abortController, stdoutAccumulator, 各种状态

2. runShellCommand({ ... }) 异步生成器
   ↑ 真正的 spawn / sandbox / 收集输出 都在这里
   每次 yield 一份 progress (output / fullOutput / elapsed / lines / bytes)
   最终 return ExecResult { stdout, code, interrupted, outputFilePath, ... }

3. 边消费生成器边调 onProgress 回调
   for { await commandGenerator.next() } until done

4. 命令结果语义解释
   interpretCommandResult(command, exitCode, stdout, '')
   → 决定 isError 是否真实
   → 给 returnCodeInterpretation 字段填语义

5. sandbox violation 注解
   SandboxManager.annotateStderrWithSandboxFailures(input.command, stdout)

6. 错误处理
   if interpretationResult.isError && !isInterrupt:
     throw new ShellError('', annotatedOutput, code, interrupted)

7. 大输出持久化
   if (result.outputFilePath && result.outputTaskId) {
     // 复制到 tool-results dir
     // 超过 64MB 截断
     persistedOutputPath = dest
     persistedOutputSize = fileStat.size
   }

8. 数据流后处理
   - stripEmptyLines
   - extractClaudeCodeHints (剥离 <claude-code-hint /> 标签)
   - isImageOutput? → resizeShellImageOutput

9. 返回 { data: Out }
```

### 关键细节 1：`_simulatedSedEdit` 短路

```ts
async call(input, toolUseContext, _canUseTool?, parentMessage?, onProgress?) {
  if (input._simulatedSedEdit) {
    return applySedEdit(input._simulatedSedEdit, toolUseContext, parentMessage)
  }
  // ... 正常流程
}
```

sed 模拟路径在 1.3 提过：permission dialog 把用户审查通过的 sed 结果塞进 `_simulatedSedEdit`，`call()` 直接走文件写入路径不走 spawn。这是 06 篇的核心机制。

### 关键细节 2：async generator + 实时 progress

`runShellCommand` 是 `AsyncGenerator`，边执行边 yield 进度：

```ts
{
  type: 'progress',
  output: string,           // 当前 chunk
  fullOutput: string,       // 至今累积
  elapsedTimeSeconds: number,
  totalLines: number,
  totalBytes?: number,
  taskId?: string,
  timeoutMs?: number,
}
```

UI 可以实时显示长命令的输出（如 `npm install` 的进度）。这与 [messages-pipeline 02 流式事件](../messages-pipeline/02-流式事件与assistant增量构造.md) 的 `createProgressMessage` 走同一套机制。

### 关键细节 3：`extractClaudeCodeHints` 零 token 侧信道

`BashTool.tsx:774-784`：

```ts
// Claude Code hints protocol: CLIs/SDKs gated on CLAUDECODE=1 emit a
// `<claude-code-hint />` tag to stderr (merged into stdout here). Scan,
// record for useClaudeCodeHintRecommendation to surface, then strip
// so the model never sees the tag — a zero-token side channel.
const extracted = extractClaudeCodeHints(strippedStdout, input.command)
strippedStdout = extracted.stripped
if (isMainThread && extracted.hints.length > 0) {
  for (const hint of extracted.hints) maybeRecordPluginHint(hint)
}
```

**侧信道协议**：第三方 CLI 工具检测到 `CLAUDECODE=1` 环境变量，可以在 stderr 里输出 `<claude-code-hint plugin="xxx" />` 标签，告诉 Claude Code "推荐用户装我这个 plugin"。Claude Code 解析标签、记录推荐、然后**剥离标签**——模型看不到。

这是个**零 token 侧信道**——工具能告诉 Claude Code 系统信息但不污染模型上下文。注释明确"a zero-token side channel"——这个设计很巧妙。

### 关键细节 4：sub-agent 不能改 cwd

```ts
const isMainThread = !toolUseContext.agentId
const preventCwdChanges = !isMainThread
```

sub-agent（有 agentId）跑命令时，`cd` 操作不会改主进程的工作目录——避免子代理把主代理的 cwd 搞乱。这是 [agent 07 工具继承与隔离](../agent/07-工具继承与隔离机制.md) 的一个具体实现点。

### 关键细节 5：cwd 越界自动重置

```ts
if (!preventCwdChanges) {
  const appState = getAppState()
  if (resetCwdIfOutsideProject(appState.toolPermissionContext)) {
    stderrForShellReset = stdErrAppendShellResetMessage('')
  }
}
```

主线程跑命令时，如果 cwd 跑到项目目录之外了（比如模型 `cd /tmp` 了），自动重置回项目根目录并在 stderr 加一条提示。这是个隐性的"防游荡"——避免模型一不小心跑到 `/etc/` 里然后 `cat` 关键文件。

## 1.8 结果转 tool_result

`BashTool.tsx:555-623` 的 `mapToolResultToToolResultBlockParam`：

```ts
mapToolResultToToolResultBlockParam({ ... }, toolUseID): ToolResultBlockParam {
  // 1. structuredContent 优先（极少见）
  if (structuredContent?.length > 0) return { content: structuredContent }

  // 2. 图片输出走 image block
  if (isImage) {
    const block = buildImageToolResult(stdout, toolUseID)
    if (block) return block
  }

  // 3. 处理 stdout：剥前导空行、trim 尾部
  let processedStdout = stdout
  if (stdout) {
    processedStdout = stdout.replace(/^(\s*\n)+/, '')
    processedStdout = processedStdout.trimEnd()
  }

  // 4. 大输出持久化：用 <persisted-output> 包装替换内容
  if (persistedOutputPath) {
    const preview = generatePreview(processedStdout, PREVIEW_SIZE_BYTES)
    processedStdout = buildLargeToolResultMessage({
      filepath: persistedOutputPath,
      originalSize: persistedOutputSize ?? 0,
      isJson: false,
      preview: preview.preview,
      hasMore: preview.hasMore,
    })
  }

  // 5. 错误消息组装
  let errorMessage = stderr.trim()
  if (interrupted) {
    if (stderr) errorMessage += EOL
    errorMessage += '<error>Command was aborted before completion</error>'
  }

  // 6. 后台任务说明
  let backgroundInfo = ''
  if (backgroundTaskId) {
    const outputPath = getTaskOutputPath(backgroundTaskId)
    if (assistantAutoBackgrounded) {
      backgroundInfo = `Command exceeded the assistant-mode blocking budget...`
    } else if (backgroundedByUser) {
      backgroundInfo = `Command was manually backgrounded by user...`
    } else {
      backgroundInfo = `Command running in background with ID: ${backgroundTaskId}...`
    }
  }

  // 7. 拼接：stdout + errorMessage + backgroundInfo
  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: [processedStdout, errorMessage, backgroundInfo].filter(Boolean).join('\n'),
    is_error: interrupted,
  }
}
```

几个细节：

- **`is_error: interrupted`**：只在用户中断时标 is_error，普通失败（exit code != 0）不标——让模型有机会基于 stderr 自己判断；
- **大输出走 `<persisted-output>`**：完整内容写盘，模型 prompt 里只放 preview + 文件路径，模型需要时用 FileRead 拿全量；
- **3 类 background 措辞**：模型需要知道是用户手动后台、还是 assistant-mode 自动后台、还是显式 `run_in_background: true`——对应不同的后续策略；
- **interrupt 标签 `<error>...`**：注意这是 BashTool 内部约定的边界标签，**不是** `<system-reminder>`。

## 1.9 几个隐性设计判断

读完 BashTool 的入口部分，有几条值得抽取：

### 1. 工具入口是个"配置壳"，业务逻辑全在外部模块

`BashTool.tsx` 1143 行看起来不少，但**核心业务全部委托出去**：

- `prompt()` → `getSimplePrompt()` in `prompt.ts`
- `isReadOnly()` → `checkReadOnlyConstraints()` in `readOnlyValidation.ts`
- `checkPermissions()` → `bashToolHasPermission()` in `bashPermissions.ts`
- `call()` → `runShellCommand` async generator → `exec()` in `Shell.ts`
- sandbox → `SandboxManager` in `sandbox-adapter.ts`

BashTool.tsx 自己只做**编排**和**结果加工**。这是一个 ToolDef 的"contract 实现"风格——每个 hook 都是声明式的，业务逻辑在专门模块里。

### 2. 防御从 schema 层就开始

`_simulatedSedEdit` 永不进模型 schema，是个**结构层防御**。模型连这个字段都看不到，自然不可能用它绕过。

这种"用 schema 隐藏 = 用结构防御"的思路，和 [prompt-injection 02 FileRead 双护栏](../prompt-injection/02-FileRead双护栏.md) 用行号前缀让攻击无法假装系统消息是同一类设计。

### 3. prompt cache 是隐形约束

`dedup()` / `normalizeAllowOnly()` / 拼 prompt 时把 per-user 字段替换成 `$TMPDIR`——这些都是为了让**不同用户、不同 session 的 prompt 共享 cache prefix**。

cache 命中带来的 token 节省非常可观（注释提到节省 150-200 tokens/request、避免 cache miss）。任何 prompt 拼接逻辑都要考虑：**这一字段在不同用户间稳定吗？**

### 4. 调用链路里多处"是否主线程"判断

`preventCwdChanges = !isMainThread`、`resetCwdIfOutsideProject` 只主线程做、`maybeRecordPluginHint` 只主线程做。

sub-agent 跑命令必须**隔离副作用**——cwd 改变、plugin 推荐都只该影响主进程。这种"主/子线程分支"在 BashTool 里至少 3 处。

### 5. 零 token 侧信道是个有创意的协议

`<claude-code-hint />` 让第三方 CLI 工具能用 stderr 传 metadata 给 Claude Code，但模型不感知。这种**"机器到机器但绕过模型"** 的协议设计在 LLM 工具里很少见，值得记一笔。

## 1.10 小结

- BashTool 是 ToolDef 的标准实现，所有真正的业务（解析/安全/权限/执行/沙箱）都委托出去；
- prompt 文本 ~5 节结构：基础说明 / 优先专用工具 / 通用指令 / sandbox 配置 / git 指令；
- inputSchema 4 个模型可见字段 + 1 个内部 `_simulatedSedEdit`（防 sed bypass）；
- outputSchema 12 字段，含 persisted output / background / 语义解释；
- `call()` 通过 async generator 流式执行 + progress 上报；
- sub-agent 路径多处隔离副作用；
- `<claude-code-hint />` 零 token 侧信道协议很巧妙。

下一篇 → [02 命令解析管线](./02-命令解析管线.md)
