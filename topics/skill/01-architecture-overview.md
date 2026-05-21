# 01. 架构总览

## 1. 核心判断

在这份代码里，skill 不是单独的运行时类型，而是 `Command` 联合类型中的一个具体形态：

- 统一抽象定义在 `restored-src/src/types/command.ts`
- skill 对应的是 `PromptCommand`
- 真正区分“这是 skill 还是普通命令”的不是 TypeScript 类型，而是 `source`、`loadedFrom`、`disableModelInvocation`、`userInvocable`、`kind` 等元数据

也就是说，项目内部的思路不是：

`Skill -> Command`

而是：

`Command = builtin command | local command | local-jsx command | prompt-based skill/command`

skill 只是 prompt 型 command 中的一类。

## 2. `Command` 模型里和 skill 直接相关的字段

`restored-src/src/types/command.ts` 里，和 skill 机制最关键的字段有这些：

| 字段 | 作用 |
| --- | --- |
| `type: 'prompt'` | 说明这个 command 的本质是“展开 prompt 后继续交给模型处理” |
| `source` | 来源，可能是 `policySettings`、`userSettings`、`projectSettings`、`plugin`、`bundled`、`mcp`、`builtin` |
| `loadedFrom` | 更细颗粒度地标记它是从 `skills`、`commands_DEPRECATED`、`plugin`、`bundled`、`mcp` 等路径进入系统 |
| `allowedTools` | skill 生效后为当前回合追加的工具权限 |
| `hooks` | 调用该 skill 时要注册的 hooks |
| `skillRoot` | skill 所在目录，用于 `CLAUDE_PLUGIN_ROOT` / `CLAUDE_SKILL_DIR` 这类资源定位 |
| `context` | `inline` 或 `fork`，决定是在当前会话展开，还是转成子 agent 执行 |
| `agent` | 当 `context === 'fork'` 时指定子 agent 类型 |
| `effort` | skill 可覆盖当前 agent 的 effort |
| `paths` | conditional skills 的路径匹配规则，只有命中相关文件才会激活 |
| `disableModelInvocation` | 为 `true` 时模型不能通过 `SkillTool` 主动调用 |
| `userInvocable` | 为 `false` 时用户不能直接 `/skill-name`，只能由模型通过 `SkillTool` 调用 |
| `getPromptForCommand()` | skill 的真正执行入口，负责把 skill 展开成模型要看到的内容块 |

## 3. `source` 和 `loadedFrom` 为什么要分开

这两个字段看起来相似，但职责不同。

### `source`

`source` 更偏“配置或能力的来源”：

- `projectSettings`
- `userSettings`
- `policySettings`
- `plugin`
- `bundled`
- `mcp`
- `builtin`

它常用于：

- UI 来源展示
- 策略判断
- telemetry
- 来源可信度判断

### `loadedFrom`

`loadedFrom` 更偏“它是通过哪条 skill/command 载入路径进入系统的”：

- `skills`
- `commands_DEPRECATED`
- `plugin`
- `bundled`
- `mcp`

它常用于：

- skill 过滤
- `/skills` 菜单分组
- SkillTool 列表判断
- prompt 里对 skill 的可见性控制

最重要的一个点是：

同样都是 `projectSettings` 来源，可能既有 `.claude/skills/.../SKILL.md` 进来的 skill，也有遗留 `.claude/commands/*.md` 进来的“命令式 skill”，二者 `source` 一样，但 `loadedFrom` 不一样。

## 4. skill 在系统里的总体流转

可以把 skill 机制看成四层：

1. 定义层
   `SKILL.md`、plugin manifest、bundled skill definition、MCP 资源
2. 装载层
   `loadSkillsDir.ts`、`loadPluginCommands.ts`、`bundledSkills.ts`、MCP loader
3. 过滤层
   `commands.ts` 里按来源、权限、描述、可调用性生成不同视图
4. 执行层
   用户 `/skill` 走 `processSlashCommand.tsx`，模型主动调用走 `SkillTool.ts`

## 5. `commands.ts` 是 skill 聚合总线

真正把 skill 融进整个命令系统的是 `restored-src/src/commands.ts`。

核心流程如下：

1. `getSkills(cwd)` 并行收集四类 skill：
   - `getSkillDirCommands(cwd)` 读取本地 `.claude/skills` 和遗留 `.claude/commands`
   - `getPluginSkills()` 读取插件 skill
   - `getBundledSkills()` 读取 bundled skill
   - `getBuiltinPluginSkillCommands()` 读取 built-in plugin skill
2. `loadAllCommands(cwd)` 再把这些 skill 和：
   - plugin commands
   - workflow commands
   - 内建命令 `COMMANDS()`
   汇总成完整命令表
3. `getCommands(cwd)` 再叠加：
   - `meetsAvailabilityRequirement()`
   - `isCommandEnabled()`
   - 本回合动态发现的 skills

这意味着 skill 从来不是“单独查表”的能力，而是先被并入整套命令集合，再从不同视角切出不同子集。

## 6. skill 相关的几个“命令视图”

`commands.ts` 里至少有三种和 skill 相关的筛选结果。

### 6.1 `getCommands(cwd)`

这是全量命令表。

它包含：

- skill
- plugin command
- workflow command
- 内建 slash command

### 6.2 `getSkillToolCommands(cwd)`

这是给模型使用 `SkillTool` 时看的可调用 skill 列表。

过滤规则大意是：

- 必须是 `type === 'prompt'`
- 不能是 `source === 'builtin'`
- 不能是 `disableModelInvocation`
- 对 plugin/MCP skill 要求有显式描述或 `whenToUse`
- `bundled`、`skills`、`commands_DEPRECATED` 会被保留

所以这个列表偏“模型可执行的 prompt skill”。

### 6.3 `getSlashCommandToolSkills(cwd)`

这个列表更像“面向系统提示或技能可见性”的 skill 子集。

它保留：

- `loadedFrom === 'skills'`
- `loadedFrom === 'plugin'`
- `loadedFrom === 'bundled'`
- 或者 `disableModelInvocation === true`

因此它既覆盖用户可见 skill，也覆盖模型专用 skill。

## 7. skill 真正执行的不是文件，而是 `getPromptForCommand()`

无论 skill 是来自本地、plugin 还是 bundled，最终都要收敛到同一个执行接口：

`getPromptForCommand(args, context): Promise<ContentBlockParam[]>`

这意味着运行时并不关心它最初是：

- `.claude/skills/review/SKILL.md`
- plugin `skills/foo/SKILL.md`
- `registerBundledSkill({...})`

只关心它能不能在当前上下文中生成一组 prompt blocks。

这也是为什么 skill 能直接复用 slash command 的整体执行框架。

## 8. `inline` 和 `fork` 是架构分叉点

skill 的运行方式由 `context` 决定：

- `inline`
  直接把 skill 展开成当前会话里的 meta user message，然后继续主会话 query
- `fork`
  调用 `prepareForkedCommandContext()` 和 `runAgent()`，把 skill 放到子 agent 里执行

这不是 UI 层差异，而是执行模型差异。

它会影响：

- skill 内容如何注入上下文
- 谁来拿工具权限
- 是否有独立 token budget
- compact 时 skill 如何按 agent 维度保留

## 9. skill 的一个重要系统性特征

在这个项目里，skill 不是“一个动作”，而是“给模型注入一段专门 workflow prompt，并临时改变本轮执行环境”的机制。

skill 影响的不只是提示词，还包括：

- 允许使用哪些工具
- 是否 fork 子 agent
- 使用哪个模型
- 使用哪个 effort
- 是否注册 hooks
- compact 后如何保留技能上下文
- telemetry 如何记录本次能力调用

所以如果只把它理解成“斜杠命令模板”，会低估这套机制的复杂度。
