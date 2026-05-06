# 09. 编一个自己的 skill — 把每个 frontmatter 字段都跑过一遍

> 这一篇做一件事：你写一个 file-based skill，逐字段对照"这个字段被前面哪一篇的哪一段代码读取"。读完之后你不只看过源码，还从作者视角走过 skill 这套机制——下次写 skill 时知道每个字段背后会让什么代码跑起来。
>
> 接 02-08 篇所有抽象讨论：这一篇是它们的"实操检验"。

---

## 1. 这一篇要做什么和不做什么

**做**：

- 在你**自己**的项目里建 `.claude/skills/<name>/SKILL.md`
- 字段一个一个加，每加一个字段就回去读对应章节的代码
- 用真实场景验证字段语义（用户 typeahead、SkillTool 列表、attachment 推送）
- 给四个对照例子：minimal、带参数、带 paths、带 fork

**不做**：

- 不教你写 bundled skill。bundled skill 需要改源码、build cli。如果你要做 bundled，01 篇讲过——核心是 `registerBundledSkill({...})`。
- 不教你写 plugin skill。plugin 在另一套 manifest 体系里。
- 不教你写 MCP skill。MCP 涉及 server 实现，不是这套教程范围。

整套示例都是 `.claude/skills/<name>/SKILL.md` 形态——`source: 'projectSettings'`、`loadedFrom: 'skills'`。

---

## 2. 准备工作

确保你在一个 git 仓库根目录下。skill 装载会向上走到 git root（03 篇 5.2 节阶段 A 讲过 `getProjectDirsUpToHome`）。

```bash
mkdir -p .claude/skills
```

每个 skill 是一个**目录**，不是单个文件——03 篇 4.1 节 `loadSkillsFromSkillsDir.ts:425-428` 明确："Single .md files are NOT supported in /skills/ directory"。

---

## 3. 例子一：最小 skill — 验证装载链路

### 3.1 写 skill

```bash
mkdir -p .claude/skills/say-hello
```

`.claude/skills/say-hello/SKILL.md`:

```markdown
---
description: Greet the user enthusiastically
---

You are now a friendly greeter. Reply to the user with a warm greeting and ask what they're working on.
```

### 3.2 验证装载

启动 Claude Code（在这个目录里），然后：

- 敲 `/sa<Tab>` 看 typeahead 有没有 `/say-hello`
- 敲 `/skills` 看菜单里有没有它（应该在 "Project skills" 分组下）
- 敲 `/say-hello` 看 prompt 是不是按 SKILL.md 的内容跑

### 3.3 这条 skill 走过哪些代码

打开 03、04、05 篇配合看：

| 步骤 | 文件:行 | 这一步对你这条 skill 做了什么 |
|---|---|---|
| 启动期，Claude Code 第一次 `getCommands(cwd)` | `commands.ts:476-517` | 触发 `loadAllCommands(cwd)` |
| 4 路并发拉 skill 来源 | `commands.ts:353-398` | `getSkillDirCommands(cwd)` 是其中一路 |
| `getSkillDirCommands` 找你的目录 | `loadSkillsDir.ts:638-803` | `getProjectDirsUpToHome('skills', cwd)` 找到 `.claude/skills` |
| 5 路装载 | `loadSkillsDir.ts:679-714` | 你的 skill 走 "project" 那一路（`source='projectSettings'`） |
| 单 skill 装载 | `loadSkillsDir.ts:407-480` | 找到 `say-hello/SKILL.md`，读，parse |
| frontmatter 解析 | `loadSkillsDir.ts:185-265` | description="Greet ..." |
| Command 构造 | `loadSkillsDir.ts:270-401` | type=prompt, source=projectSettings, loadedFrom=skills, baseDir=`<...>/.claude/skills/say-hello`, hasUserSpecifiedDescription=true |
| realpath 去重 | `loadSkillsDir.ts:728-763` | 你只有一份，不冲突 |
| paths 分流 | `loadSkillsDir.ts:771-790` | 你没写 paths，进 unconditional |
| 进总表 | `commands.ts:460-468` | 拼接顺序 ...skillDirCommands... ← 你在这 |
| typeahead 显示 | `commandSuggestions.ts:36` | 过滤 `!cmd.isHidden` 通过 |
| `/skills` 菜单展示 | `SkillsMenu.tsx:234-236` | `loadedFrom === 'skills'` 通过 |
| 用户敲 `/say-hello` | `processSlashCommand.tsx:309` | 解析 → `findCommand` 命中 |
| inline prompt 注入 | `processSlashCommand.tsx:827-925` | `addInvokedSkill('say-hello', ...)`、metadata、isMeta user message |
| `getPromptForCommand` 闭包跑 | `loadSkillsDir.ts:344-399` | 注入 base directory 头部，无参数替换，无 shell block，返回 text |
| 主 query 循环消费 | (REPL) | 模型在下一轮看到 SKILL_BODY |

#### 思考题

> `description` 字段没写：删掉它，重启 Claude Code，再敲 `/say-hello`。用户看到的 description 是什么？

（答：fallback 是 markdown 正文第一行—— `extractDescriptionFromMarkdown(content, 'Skill')`（03 篇 3.3 节 (a) 关键观察提到的"两级 fallback"）。`hasUserSpecifiedDescription` 会变成 false——但这一变化对当前 skill 没影响（plugin/MCP 才需要它放行）。）

---

## 4. 例子二：带参数 skill — 验证 `${args}` 替换

### 4.1 写 skill

```bash
mkdir -p .claude/skills/review-pr
```

`.claude/skills/review-pr/SKILL.md`:

```markdown
---
description: Review a GitHub PR with attention to security
when_to_use: When the user asks to review a PR, especially for security concerns
argument-hint: <pr-number>
arguments:
  - pr_number
---

# PR Security Review

The user has asked you to review PR ${pr_number}.

Steps:
1. Use `gh pr view ${pr_number}` to get the PR details
2. Use `gh pr diff ${pr_number}` to see changes
3. Look for security issues:
   - SQL injection risk
   - Hardcoded secrets
   - Auth bypasses
   - Unsafe deserialization
4. Report findings with severity levels

Skill base directory: ${CLAUDE_SKILL_DIR}
```

### 4.2 验证

- 敲 `/review-pr` 不带参数 → 看 `${pr_number}` 在 prompt 里是什么样
- 敲 `/review-pr 123` → 看 prompt 里 `${pr_number}` 替换成 `123`
- 看 typeahead 提示符是 `<pr-number>`（来自 `argument-hint`）

### 4.3 字段对照

| 字段 | 解析在 | 用在哪 | 注意 |
|---|---|---|---|
| `when_to_use` | `loadSkillsDir.ts:252` | `getSkillToolCommands`/`getSlashCommandToolSkills` 进表必备之一；attachment 系统给模型看 | 注意是下划线分隔 `when_to_use`，不是 `whenToUse`（YAML key），但运行时字段名变成 camelCase |
| `argument-hint` | `loadSkillsDir.ts:245-248` | typeahead 显示灰色提示 | 也是 dash 分隔 |
| `arguments` | `loadSkillsDir.ts:249-251` (`parseArgumentNames`) | `argumentNames` 数组，传给 `substituteArguments` | 可以是字符串或数组 |
| `${pr_number}` 替换 | `loadSkillsDir.ts:349-354` | 闭包内 `substituteArguments(finalContent, args, true, argumentNames)` | 参数替换是文本级（不是 AST） |
| `${CLAUDE_SKILL_DIR}` | `loadSkillsDir.ts:359-363` | 替换成 baseDir | Win 上 `\` → `/` |

#### 思考题

> 如果用户敲 `/review-pr "feat: add auth"` 但 skill 的 `arguments` 只声明了 `pr_number`，会发生什么？是按位置赋值还是按命名？

（提示：参考 `parseArgumentNames` 和 `substituteArguments` 的实现细节。简单的"位置参数 + 单 arg 字符串"模型——args 整体作为单个值传入，按 `${pr_number}` 占位符替换。复杂参数解析需要手写 prompt 内容指导模型。）

---

## 5. 例子三：带 paths 的 conditional skill

### 5.1 写 skill

```bash
mkdir -p .claude/skills/payments-helper
```

`.claude/skills/payments-helper/SKILL.md`:

```markdown
---
description: Helper for working with payments code
when_to_use: When working in src/payments or anything stripe-related
paths:
  - 'src/payments/**'
  - '**/*stripe*'
---

# Payments Code Conventions

When you edit payments code, follow these rules:

1. ALL money amounts must be in cents (integer), never decimal dollars.
2. Stripe customer IDs follow the pattern `cus_*`.
3. Charges must be idempotent — use the `idempotency_key` field.
4. Never log full credit card numbers, only last 4.
5. Test mode keys start with `sk_test_`, prod with `sk_live_`.
```

### 5.2 验证

启动 Claude Code，然后试这个序列：

```text
session 开始
↓
看 /skills 菜单 → payments-helper **不在** 列表里
↓
让 Claude Read 一个无关文件（比如 README.md）
↓
回看 /skills 菜单 → 仍然不在
↓
让 Claude Read src/payments/charge.ts
↓
观察：下一个 turn 模型会看到一条 attachment 提示新 skill
↓
回看 /skills 菜单 → payments-helper 出现了
```

### 5.3 字段对照

打开 07 篇 4.1 + 4.2 节配合看：

| 行为 | 对应代码 |
|---|---|
| 装载时被抽走 | `loadSkillsDir.ts:771-790` 把它放进 `conditionalSkills` map |
| 不进 `getCommands` 默认结果 | `loadSkillsDir.ts:802` `return unconditionalSkills` |
| FileRead 触发激活 | `FileReadTool.ts:590` 调 `activateConditionalSkillsForPaths` |
| 路径匹配 | `loadSkillsDir.ts:1012` `ignore().add(skill.paths)` |
| 命中后搬到 dynamicSkills | `loadSkillsDir.ts:1031-1033` |
| 永久标记激活 | `loadSkillsDir.ts:1033` `activatedConditionalSkillNames.add` |
| signal emit | `loadSkillsDir.ts:1054` `skillsLoaded.emit()` |
| 命令视图 cache 浅清 | `skillChangeDetector.ts:97` `clearCommandMemoizationCaches()` |
| 下次 turn skill_listing 推送 | `attachments.ts:2680-2751` |

#### 验证小细节

`paths: 'src/payments/**'` 中的 `/**` 后缀会被 `parseSkillPaths`（`loadSkillsDir.ts:159-178`）**剥掉**——`/**` 在 `ignore` 库语义里反而限制只匹配内部，不匹配 path 自身。剥掉后变成 `'src/payments'`，匹配整个目录树。

`'**/*stripe*'` 则保留——它是 glob，不是路径前缀。

#### 思考题

> 如果 `paths: '**'`（所有路径），这个 skill 还是 conditional 吗？

（答：不是。`parseSkillPaths` 第 173-175 行：`if (patterns.length === 0 || patterns.every(p => p === '**'))` → return undefined。**"全 match" 等价于"无限制"**——直接进 unconditional。）

---

## 6. 例子四：带 `allowed-tools` 的 skill — 验证临时权限

### 6.1 写 skill

```bash
mkdir -p .claude/skills/git-status-helper
```

`.claude/skills/git-status-helper/SKILL.md`:

```markdown
---
description: Show git status with extra annotations
allowed-tools:
  - 'Bash(git status:*)'
  - 'Bash(git diff:*)'
  - 'Bash(git log:*)'
---

Show me a comprehensive git status with annotations:

1. Run `git status` to see modified files.
2. For each modified file, run `git diff <file> | head -30` to see top changes.
3. Run `git log --oneline -5` to show recent commits.
4. Summarize: how many files changed, what kinds (test/source/config).
```

### 6.2 验证

- 敲 `/git-status-helper`
- 模型应该可以**直接**跑 `git status`、`git diff README.md`、`git log` —— **不弹权限确认**
- 但是模型要跑 `npm install` 时还会弹权限（不在 allowedTools 里）
- 再敲个 `/clear` 然后让 Claude 调 `git status`——会弹权限（`allowedTools` 是 per-turn 的）

### 6.3 字段对照

打开 03、05、06 篇配合看：

| 阶段 | 代码 |
|---|---|
| 解析 | `loadSkillsDir.ts:242-244` `parseSlashCommandToolsFromFrontmatter(frontmatter['allowed-tools'])` |
| 装入 Command | `loadSkillsDir.ts:322` 字段 `allowedTools` |
| 用户调用：进 attachment | `processSlashCommand.tsx:887` `additionalAllowedTools = parseToolListFromCLI(command.allowedTools ?? [])` |
| attachment 注入 | `processSlashCommand.tsx:910-914` `command_permissions` attachment |
| 主 REPL 消费 | 主循环把 `command_permissions` 合并进 `toolPermissionContext.alwaysAllowRules.command` |
| 模型调用：safe skill 判定 | `SkillTool.ts:529-538` 因为 `allowedTools` 非空 → **不算** safe → 弹权限 |
| 用户允许后 | `SkillTool.ts:775-806` `contextModifier` 把 `allowedTools` 写回 `alwaysAllowRules.command` |

#### 关键观察

**(a) skill 加了 `allowed-tools` 之后**，从 SkillTool 视角它**不再是 safe skill**——会弹权限。从用户 slash 视角直接放行（用户敲了就是同意）。

这是 06 篇 5.3 节讨论的 `SAFE_SKILL_PROPERTIES` allowlist 行为——`allowedTools` 不在 allowlist 里。

**(b) `allowed-tools` 的字符串语法**

支持几种格式：

- `'Bash'` — 全部 Bash 调用
- `'Bash(git:*)'` — 只允许 git 子命令
- `'Bash(git status:*)'` — 只允许 git status 系列
- `'Read'` — 全部 Read
- `'mcp__server__tool'` — 特定 MCP 工具

由 `parseSlashCommandToolsFromFrontmatter` 解析。

#### 思考题

> 模型通过 SkillTool 调 git-status-helper，弹了权限。用户允许之后，本 turn 模型能直接跑 `git status`。下个 turn 还能直接跑吗？为什么？

（答：取决于用户允许时点了哪个按钮。如果选 "Yes, just this time"——只本 turn。如果选 "Always allow"——存进永久 rules，下次还有效。看 `checkPermissions` 里的 `suggestions`（行 540-567）——它给了两条建议规则，让用户保存。）

---

## 7. 例子五：fork 模式 skill

### 7.1 写 skill

```bash
mkdir -p .claude/skills/large-refactor
```

`.claude/skills/large-refactor/SKILL.md`:

```markdown
---
description: Run a large refactoring task in a sub-agent
when_to_use: When the user wants a large multi-file refactor that should not pollute the main session
context: fork
agent: general-purpose
allowed-tools:
  - 'Read'
  - 'Edit'
  - 'Bash(rg:*)'
  - 'Glob'
  - 'Grep'
---

You are doing a large refactoring task in an isolated sub-agent.

Steps:
1. Use Glob/Grep to find all relevant files.
2. Plan the refactoring before making changes.
3. Edit files in batches.
4. After completion, return a structured summary:
   - Files changed
   - Lines added/removed
   - Any tests affected
```

### 7.2 验证

- 敲 `/large-refactor "rename all snake_case to camelCase in src/"` 看模型怎么响应
- 注意它会**在子 agent 里执行**——主会话只看到一条 "Skill completed (forked)" + result 摘要
- 主会话的 conversation 不会被 refactor 过程的几百条 tool_use 污染

### 7.3 字段对照

打开 05、06 篇 fork 部分：

| 阶段 | 代码 |
|---|---|
| 解析 `context: fork` | `loadSkillsDir.ts:260` `executionContext: frontmatter.context === 'fork' ? 'fork' : undefined` |
| 解析 `agent` | `loadSkillsDir.ts:261` |
| 用户调用走 fork 分支 | `processSlashCommand.tsx:715-720` `if (command.context === 'fork') executeForkedSlashCommand(...)` |
| 模型调用走 fork 分支 | `SkillTool.ts:622-632` `executeForkedSkill(...)` |
| 准备子 agent context | `forkedAgent.ts:prepareForkedCommandContext` (需自己 grep) |
| 子 agent 跑 | `runAgent` 里跑独立循环 |
| 子 agent invokedSkills 隔离 | `bootstrap/state.ts` 按 (name, agentId) 复合键 |
| 跑完清子 agent invokedSkills | `SkillTool.ts:287` `clearInvokedSkillsForAgent(agentId)` |

#### 关键观察

**(a) fork 模式不复用 `processPromptSlashCommand`**

参考 06 篇 3.2 节——`executeForkedSkill` 直接调 `prepareForkedCommandContext + runAgent`，不走主会话的 prompt 注入路径。

**(b) `agent` 字段决定子 agent 类型**

值是 agent 定义的名字。`general-purpose` 是 Claude Code 内置的通用子 agent。其它选项参考 `restored-src/src/cli/agents/` 或者 plugin 提供的 agent。

**(c) `allowed-tools` 在 fork 模式下传给子 agent**

子 agent 启动时用这些工具权限——它和主会话隔离。所以"sub-agent 能跑 Bash(rg:*) 不影响主会话"。

#### 思考题

> 主会话调用 fork skill，skill 内部又触发 SkillTool 调用一个**inline** skill。这个 inline skill 的 prompt 注入到哪？是注入到主会话还是子 agent 会话？

（答：子 agent。因为 SkillTool.call 拿的 context 已经是 forked context（modifiedGetAppState、子 agent 的 toolPermissionContext）。inline skill 的 prompt 通过 `processPromptSlashCommand` 注入"当前 context 的会话"——也就是子 agent。所以 fork → inline 嵌套是干净的：嵌套的所有 skill 都待在 fork 域内。）

---

## 8. 例子六：带 hooks 的 skill

### 8.1 写 skill

```bash
mkdir -p .claude/skills/audit-mode
```

`.claude/skills/audit-mode/SKILL.md`:

```markdown
---
description: Run in audit mode — log all tool uses
hooks:
  PreToolUse:
    - matcher: '*'
      hooks:
        - type: command
          command: 'echo "[audit] About to use tool" >> ~/audit.log'
  PostToolUse:
    - matcher: '*'
      hooks:
        - type: command
          command: 'echo "[audit] Tool finished" >> ~/audit.log'
---

You are in audit mode. Every tool you use will be logged to ~/audit.log.
Proceed with the user's request normally.
```

### 8.2 验证

- 敲 `/audit-mode` 然后让 Claude 跑一个普通任务
- 看 `~/audit.log` 应该多了几行
- 退出 Claude Code 重启，hooks 应该不在了（hooks 是 session-scoped）

### 8.3 字段对照

打开 02、05 篇 hooks 部分：

| 阶段 | 代码 |
|---|---|
| YAML 解析 | `loadSkillsDir.ts:259` `parseHooksFromFrontmatter` |
| zod 校验 | `loadSkillsDir.ts:144` `HooksSchema().safeParse` |
| 装入 Command | `loadSkillsDir.ts:342` 字段 `hooks` |
| 注册策略门 | `processSlashCommand.tsx:870-873` `isRestrictedToPluginOnly('hooks') || isSourceAdminTrusted(command.source)` |
| 实际注册 | `registerSkillHooks(setAppState, sessionId, hooks, name, skillRoot)` |
| Skill is not safe → 弹权限 | `SkillTool.ts:875-908` `SAFE_SKILL_PROPERTIES` 不含 `hooks` |

#### 关键观察

**(a) hooks 默认让 skill 变成 "non-safe"**

`hooks` 不在 `SAFE_SKILL_PROPERTIES` 里——任何带 hooks 的 skill 模型调用时都要弹权限确认。

**(b) plugin-only 模式下 user skill 的 hooks 不会注册**

这个 skill 的 source 是 `projectSettings`。如果用户配了 `restrictPluginOnly` 策略，且 hooks 锁住了，注册会被跳过——skill 内容仍然加载但 hooks 不生效。

**(c) hook command 的环境变量**

hook 命令运行时会有 `CLAUDE_PLUGIN_ROOT`（指向 skillRoot）、`CLAUDE_SESSION_ID` 等环境变量。这是 02 篇 5.6 节 `skillRoot` 字段的一个用途。

---

## 9. 例子七：带 reference files 的 skill

bundled skill 有 `files: { 'examples/foo.md': '...' }` 的能力（01 篇 3.4 节）。**file-based skill 可以直接放文件在目录里**——用 `${CLAUDE_SKILL_DIR}` 引用。

### 9.1 写 skill

```bash
mkdir -p .claude/skills/code-review/examples
```

`.claude/skills/code-review/SKILL.md`:

```markdown
---
description: Review code following project conventions
when_to_use: When the user asks to review code or check style
---

# Code Review

Review the user's code with these conventions in mind.

Reference examples are in: `${CLAUDE_SKILL_DIR}/examples/`

Read these as needed:
- `${CLAUDE_SKILL_DIR}/examples/good-pattern.md` — patterns we like
- `${CLAUDE_SKILL_DIR}/examples/bad-pattern.md` — patterns to avoid
```

`.claude/skills/code-review/examples/good-pattern.md`:

```markdown
# Good patterns

- Pure functions over mutating helpers
- Small composable units
- Named constants over magic numbers
```

`.claude/skills/code-review/examples/bad-pattern.md`:

```markdown
# Bad patterns

- Deeply nested conditionals (>3 levels)
- Functions with side effects masquerading as pure
- Comments explaining WHAT instead of WHY
```

### 9.2 验证

- 敲 `/code-review`
- 模型应该看到 prompt 里 `${CLAUDE_SKILL_DIR}` 已被替换成绝对路径
- 模型可以用 Read 工具读那两个 examples
- 模型基于 examples 给你 review

### 9.3 字段对照

| 阶段 | 代码 |
|---|---|
| baseDir 设置 | `loadSkillsDir.ts:466` `baseDir: skillDirPath` |
| 注入 base directory 头部 | `loadSkillsDir.ts:345-347` `Base directory for this skill: <dir>\n\n${markdownContent}` |
| `${CLAUDE_SKILL_DIR}` 替换 | `loadSkillsDir.ts:359-363` |

模型看到的最终 prompt 是：

```
Base directory for this skill: /your/project/.claude/skills/code-review

# Code Review

Review the user's code with these conventions in mind.

Reference examples are in: `/your/project/.claude/skills/code-review/examples/`

Read these as needed:
- `/your/project/.claude/skills/code-review/examples/good-pattern.md` — ...
- `/your/project/.claude/skills/code-review/examples/bad-pattern.md` — ...
```

模型自然会用 Read 工具读这两个文件。

---

## 10. 例子八：带 inline shell block 的 skill

### 10.1 写 skill

```bash
mkdir -p .claude/skills/repo-stats
```

`.claude/skills/repo-stats/SKILL.md`:

```markdown
---
description: Show repository statistics
shell: bash
---

# Repo Stats

Current branch: !`git branch --show-current`

Last commit: !`git log -1 --format=%h\ %s`

Modified files: !`git status --short | wc -l`

File counts:
!`find . -name '*.ts' -not -path '*/node_modules/*' | wc -l` TypeScript files
!`find . -name '*.tsx' -not -path '*/node_modules/*' | wc -l` TSX files
```

### 10.2 验证

- 敲 `/repo-stats`
- 模型看到的 prompt 里 `!`git ...`` 已经被**实际执行结果**替换
- 这些 shell 是在 skill 展开时（runtime）跑的，不是在 markdown 里

### 10.3 字段对照

| 阶段 | 代码 |
|---|---|
| `shell: bash` 解析 | `loadSkillsDir.ts:263` `parseShellFrontmatter(frontmatter.shell, resolvedName)` |
| 闭包安全门 | `loadSkillsDir.ts:374` `if (loadedFrom !== 'mcp')` ← 通过 |
| 实际执行 | `loadSkillsDir.ts:375` `executeShellCommandsInPrompt(...)` |
| 临时权限上下文 | `loadSkillsDir.ts:379-391` 注入 `allowedTools` 到 `alwaysAllowRules.command` |

#### 关键安全观察

**(a) MCP skill 永远不能跑 inline shell**

03 篇 3.4 节关键观察 (d) 提过——`loadedFrom !== 'mcp'` 是硬安全门。MCP 是远程不可信，永远不能跑本地 shell。

**(b) shell 执行时使用 skill 的 `allowed-tools` 当临时权限**

这是个**临时上下文 patching**——shell block 跑的时候 `getAppState()` 被重写，让 `alwaysAllowRules.command` 等于 skill 的 `allowedTools`。这样 shell 命令也能复用 skill 的工具权限。

如果 skill 没声明 `allowed-tools`，这里就是空 list——shell 还是能跑（它走的是 `Bash` 工具，本身有自己的权限策略）但没有"额外放行"。

**(c) 别滥用 inline shell**

shell block 在 skill 加载时**同步**跑——会拖慢 prompt 注入。如果 shell 命令慢（比如 `find` 大目录），用户会觉得 `/skill` 卡顿。复杂查询应该让模型用 Bash 工具自己跑。

---

## 11. 综合对照表：每个字段对应到哪一段代码

整套 SKILL.md frontmatter 字段 → 代码位置对照表，留档：

| frontmatter 字段 | YAML key | 解析在 (`loadSkillsDir.ts`) | 影响什么 |
|---|---|---|---|
| name (display) | `name` | `parseSkillFrontmatterFields:238` | `userFacingName()` |
| description | `description` | `parseSkillFrontmatterFields:208-214` | typeahead、SkillTool 列表过滤 |
| when to use | `when_to_use` | `parseSkillFrontmatterFields:252` | system prompt skill_listing |
| 允许工具 | `allowed-tools` | `parseSkillFrontmatterFields:242-244` | 本 turn `command_permissions` |
| 参数列表 | `arguments` | `parseSkillFrontmatterFields:249-251` | `${name}` 替换 |
| 参数提示 | `argument-hint` | `parseSkillFrontmatterFields:245-248` | typeahead 灰色提示 |
| 模型 | `model` | `parseSkillFrontmatterFields:221-226` | 本 turn `mainLoopModel` |
| effort | `effort` | `parseSkillFrontmatterFields:228-235` | 本 turn `effortValue` |
| 用户可调用 | `user-invocable` | `parseSkillFrontmatterFields:216-219` | `processSlashCommand` 拦截 |
| 模型不可调用 | `disable-model-invocation` | `parseSkillFrontmatterFields:255-257` | `SkillTool.validateInput` 拦截 |
| hooks | `hooks` | `parseSkillFrontmatterFields:259` (`parseHooksFromFrontmatter`) | session-scoped lifecycle hooks |
| 执行上下文 | `context` | `parseSkillFrontmatterFields:260` | `inline` vs `fork` 分流 |
| fork agent 类型 | `agent` | `parseSkillFrontmatterFields:261` | `executeForkedSkill` 子 agent 类型 |
| 路径条件 | `paths` | `parseSkillPaths:159-178` | 装载时延迟曝光 |
| shell 类型 | `shell` | `parseSkillFrontmatterFields:263` | inline shell block 用 bash 还是 powershell |
| 版本 | `version` | `parseSkillFrontmatterFields:253` | telemetry / hook 兼容 |

特殊变量替换（不是 frontmatter，是正文内的占位符）：

| 占位符 | 替换在 (`loadSkillsDir.ts`) | 替换成 |
|---|---|---|
| `${args}` (按名) | `:349-354` `substituteArguments` | 用户传入参数 |
| `${CLAUDE_SKILL_DIR}` | `:359-363` | skill 目录绝对路径 |
| `${CLAUDE_SESSION_ID}` | `:366-369` | 当前 session 的 UUID |
| `` !`cmd` `` shell block | `:374-396` | 命令实际执行结果（非 MCP only） |

---

## 12. 这一篇你应该带走的几样东西

读到这里，你写了至少 4 个例子（包括对照），你应该能：

1. 不查文档凭记忆写一个最小 SKILL.md（name 取目录名，description）
2. 写带参数的 skill 并知道参数是文本替换，不是强类型
3. 写 paths skill 并理解它在装载时**不进**默认列表，靠 FileRead 触发激活
4. 写 fork skill 并知道它走 `executeForkedSlashCommand` / `executeForkedSkill`，不走主会话注入
5. 写带 `allowed-tools` 的 skill 并知道它会让 SkillTool 弹权限（除非用户/项目级 allow rule 已配置）
6. 解释为什么 hooks 字段使 skill 变成 non-safe
7. 在 SKILL.md 内用 `${CLAUDE_SKILL_DIR}` + reference files 替代 bundled skill 的 `files` 机制
8. 安全 SKILL 内嵌 shell block，但知道 MCP skill 永远不能这样

---

## 13. 下一步

打开 `10-常见疑惑与陷阱.md`。

我们已经读完了所有源码、动手写过例子。最后一篇集中讨论那些"非显然设计"——读源码时容易问"为什么这里要这么写"的小细节，每个对应到代码里某行。这是整套教程的"答疑"。
