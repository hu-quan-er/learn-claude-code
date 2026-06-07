# 01. Command — 命令的统一抽象

> 这一篇做一件事：把 `restored-src/src/types/command.ts` 全部 217 行**作为 model 本身**讲透——每个字段是什么、为什么存在、和谁配合工作、设计哲学是什么。
>
> 与 `topics/skill/02-类型系统先行.md` 的差别：那篇是从"读懂 skill 模块"的角度切入；这一篇是从"理解 Command 这个 model 的整体设计"切入。重叠是必然的，但视角不同——这一篇会更聚焦"为什么 Command 要长这样"，把它放在所有 model 之间的关系里看。

---

## 1. Command 在整套 model 里的位置

`Command` 是 Claude Code 里"**所有可被用户输入或模型调用的能力单元**"的统一抽象。简单说：

- 用户敲 `/foo` → 找一个 `Command` 来跑
- 模型调 `SkillTool({skill: 'foo'})` → 同样找 `Command` 来跑
- typeahead 自动补全 → 列出所有 `Command`
- `/help` 列表 → 列出所有 `Command`
- `/skills` 菜单 → 列出 `Command` 中"是 skill"的那一族
- attachment 系统给模型推送 skill_listing → 还是 `Command` 中"是 skill"的那一族

**一切以"slash 名"或"skill 名"出现的能力，都是 Command。**

这意味着 Command 至少要满足：

- **多类来源**（内建 / file-based / plugin / bundled / MCP）能用同一形状装入
- **多种执行模型**（注入 prompt / 跑本地代码 / 渲染 React UI）能在同一类型里区分
- **多种消费方**（typeahead / SkillTool / `/skills` 菜单）能从同一张表切出不同视图
- **多套元数据**（权限、模型覆盖、生命周期 hooks、reference 资源等）能挂在同一个对象上

`types/command.ts` 这个 217 行的文件就是回答"怎么用 TypeScript 表达上面四点"的。

---

## 2. 整体形状：CommandBase + 三个具体类型

打开 `restored-src/src/types/command.ts`。最关键的是文件末尾这两行（`205-206`）：

```ts
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

整个 model 的骨架就这一行能说清：

```
Command = (所有命令共享的元数据) & (三类执行形态之一)
                ↑                          ↑
              CommandBase              type discriminator
              (155-203 行)             - PromptCommand   (25-57)
                                       - LocalCommand    (74-78)
                                       - LocalJSXCommand (144-152)
```

为什么要这么设计？

**不能把所有字段都放进 CommandBase**——`PromptCommand` 有 `getPromptForCommand` 函数，`LocalCommand` 有 `load + call`，`LocalJSXCommand` 有 `load + call(onDone, ctx, args)`，三者**执行入口的签名根本不同**，强行合并要么用 `any` 要么大量 optional——TypeScript 类型保护就崩了。

**也不能把所有字段都放进具体类型**——`name`、`description`、`isHidden`、`availability`、`isEnabled` 这些是**所有命令都有**的，每个具体类型重写一遍是冗余。

`& union` 的设计同时解决两个问题：共享部分共享，独有部分按 `type` 字段判别。这是 TypeScript discriminated union 的标准模式，但用得很彻底。

---

## 3. CommandBase（155-203）— 所有命令共享的"身份卡 + 可见性 + 元数据"

```ts
export type CommandBase = {
  availability?: CommandAvailability[]
  description: string
  hasUserSpecifiedDescription?: boolean
  isEnabled?: () => boolean
  isHidden?: boolean
  name: string
  aliases?: string[]
  isMcp?: boolean
  argumentHint?: string
  whenToUse?: string
  version?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  loadedFrom?:
    | 'commands_DEPRECATED' | 'skills' | 'plugin'
    | 'managed' | 'bundled' | 'mcp'
  kind?: 'workflow'
  immediate?: boolean
  isSensitive?: boolean
  userFacingName?: () => string
}
```

按职责分组看：

### 3.1 身份卡（命令是谁）

| 字段 | 类型 | 含义 |
|---|---|---|
| `name` | `string` | 唯一查找键。`/verify` 通过它命中。 |
| `aliases` | `string[]` | 别名。一个命令可以被多个名字找到（`/v` → verify）。 |
| `userFacingName` | `() => string` | 显示名（默认等于 `name`）。Plugin skill 内部 name 是 `my-plugin:review` 但 UI 显示 `review`。 |
| `version` | `string` | 命令自己声明的版本。telemetry 和 hook 兼容性用。 |

`name` 和 `userFacingName` 的拆分很有意思——它解决了 **"内部唯一标识"和"用户友好显示"是两件不同的事**这个常见问题。如果 plugin 用 `name = 'review'` 直接命名，多个 plugin 就会冲突；用 `name = 'plugin:review'` 又难看。这个分离让两者都能要。

### 3.2 描述（命令做什么）

| 字段 | 类型 | 含义 |
|---|---|---|
| `description` | `string` | **必填**。一行简短描述。给用户看（typeahead）。 |
| `hasUserSpecifiedDescription` | `boolean` | `description` 是不是作者明确写的（vs loader 从 markdown 首行抽取的）。 |
| `whenToUse` | `string` | 详细使用场景。给**模型**看（system prompt 里的 skill_listing）。 |
| `argumentHint` | `string` | 参数提示，typeahead 灰色显示。 |

**`description` vs `whenToUse` 的拆分**：一个给人，一个给模型。`description` 通常 1 行；`whenToUse` 可以多行，可能比较啰嗦。这种"按受众分字段"在所有 LLM 应用里都该模仿——人和模型对信息的偏好真不一样。

**`hasUserSpecifiedDescription` 的存在**反映了 loader 的现实：file-based skill 可能 frontmatter 没写 description，loader 会从 markdown 正文第一行抽。这种"猜的"description 在某些过滤里要被排除（比如 plugin/MCP 进 SkillTool 列表必须有显式描述）。这个布尔标记让"猜的"和"明写的"能区分。

### 3.3 可见性 / 可调用性 — 五个字段，五个维度

这一组是 CommandBase 里**最容易混的**。一定要逐个拆开理解。

| 字段 | 决定什么 | 静态/动态 |
|---|---|---|
| `disableModelInvocation` | 模型能否通过 SkillTool 调用 | 静态 |
| `userInvocable` | 用户能否 `/name` 直接调用 | 静态 |
| `isHidden` | 是否在 typeahead/`/help` 显示 | 静态 |
| `isEnabled` | 当前是否启用（feature flag、env 等）| **动态**（函数） |
| `availability` | 哪种 auth provider 下可用 | 半动态（auth 切换会变） |

#### `disableModelInvocation` (`true` 表示模型不能调)

典型例子：`skillify`——用户主动发起生成 skill，模型不应自己决定要生成。

读取点：`SkillTool.ts:412-418` 的 `validateInput`——直接拒绝。

#### `userInvocable` (`false` 表示用户不能直接 `/name`)

读取点：`processSlashCommand.tsx:512-525`。如果用户敲了，返回温和的 "This skill can only be invoked by Claude" 提示。

#### `isHidden` 跟 `userInvocable` 看似重复，但维度不同

```
userInvocable: false → 行为约束（敲了无效）
isHidden: true        → UI 约束（不在补全里）
```

它们经常**同步**（看 `bundledSkills.ts:95` 的 `isHidden: !(definition.userInvocable ?? true)`）——不允许调用的就别在补全里出现。但**允许独立**，因为某些场景需要"允许调用但补全不显示"（deprecated 但还能用、内部诊断命令等）。

#### `isEnabled` 是函数，不是 boolean

```ts
isEnabled?: () => boolean
```

每次 `getCommands()` 都会**实时调用**这个函数（看 `commands.ts:214-216`）。原因：feature flag 可能远程刷新（GrowthBook），不能 memoize。

典型例子：`isEnabled: () => isKairosCronEnabled()`——按 GrowthBook 决定 `/loop` 是否可见。

#### `availability` 是 auth/provider 维度

```ts
type CommandAvailability = 'claude-ai' | 'console'
availability?: CommandAvailability[]
```

声明这个命令针对哪种 auth 类型。`commands.ts:417-443` 的 `meetsAvailabilityRequirement` 用它过滤。注释 159-167 总结得很到位：

```
This is separate from `isEnabled()`:
  - `availability` = who can use this (auth/provider requirement, static)
  - `isEnabled()`  = is this turned on right now (GrowthBook, platform, env vars)
```

#### 五个字段的设计哲学

为什么不能合并？因为它们解决**不同决策点的不同问题**：

```
用户敲 /name 时:    userInvocable
模型调 SkillTool:    disableModelInvocation
typeahead 渲染:     isHidden
所有视图过滤前提:   isEnabled
auth 相关过滤:      availability
```

每个消费方只看自己关心的字段。合并就要在所有消费方写"如果是这个值的某种组合就 ..."——逻辑膨胀。

#### 思考题

> 这 5 个字段**全部都设成最严格**：`disableModelInvocation: true`、`userInvocable: false`、`isHidden: true`、`isEnabled: () => false`、`availability: ['claude-ai']`（用户不是 claude.ai）。
> 这个 Command 还会被装入命令表吗？过滤后还能找到吗？为什么有这种"强排除"的需求？

（提示：装载层不看这 5 个字段——所有 Command 都进 `loadAllCommands` 总表。`getCommands()` 才过滤。所以总表里还在，但任何视图都拿不到它。这种"装得到 + 用不到"的状态在动态发现 / hot reload 中间过渡阶段有意义。）

### 3.4 来源标签 `loadedFrom`

```ts
loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
```

回答"这个命令通过**哪条 loader 路径**进来的"。

- `bundled` — 代码注册（`registerBundledSkill`）
- `skills` — 磁盘 `.claude/skills/<name>/SKILL.md`
- `commands_DEPRECATED` — 旧式 `.claude/commands/...`
- `plugin` — plugin manifest 装的
- `mcp` — MCP server 提供的 skill
- `managed` — 受策略管理

**这个字段和 `PromptCommand.source` 的关系**留到 6 节讲。

### 3.5 杂项标志位

| 字段 | 用途 |
|---|---|
| `isMcp` | 这是 MCP-backed 命令（MCP prompt，不是 MCP skill） |
| `kind: 'workflow'` | 标识 WorkflowTool 创建的命令，autocomplete 加角标 |
| `immediate` | 不等"模型停下"就可执行（`/exit`、`/clear` 这类不能等的） |
| `isSensitive` | args 在 conversation 历史里要脱敏（密码、token） |

这五个都是窄场景标志位，放在 base 而不是某个具体类型里——任何 Command 都可能需要其中一个。

---

## 4. 三类具体执行形态

`Command` 的 union 部分。type 字段是判别器。

### 4.1 PromptCommand（25-57 行）— skill 用的就是这个

```ts
export type PromptCommand = {
  type: 'prompt'
  progressMessage: string
  contentLength: number
  argNames?: string[]
  allowedTools?: string[]
  model?: string
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  pluginInfo?: { pluginManifest: PluginManifest; repository: string }
  disableNonInteractive?: boolean
  hooks?: HooksSettings
  skillRoot?: string
  context?: 'inline' | 'fork'
  agent?: string
  effort?: EffortValue
  paths?: string[]
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

**它的本质**：执行 = 把内容展开成一段 prompt 给模型看。

#### 字段分组

**执行入口**：

- `getPromptForCommand(args, ctx)` — 唯一执行点，返回 `ContentBlockParam[]`（Anthropic SDK 消息块）

**修饰本轮执行环境**（这是 PromptCommand 区别于普通 prompt 模板的根本）：

- `allowedTools` — 临时追加的工具权限
- `model` — 切换本轮模型
- `effort` — 切换 thinking 预算
- `context` / `agent` — inline 还是 fork（fork 时跑哪个子 agent）
- `hooks` — 注册 lifecycle hooks
- `paths` — 条件激活的路径模式

**资源定位**：

- `skillRoot` — skill 资源根目录（`${CLAUDE_SKILL_DIR}` / `${CLAUDE_PLUGIN_ROOT}` 替换基准）

**身份补充**（这些不在 CommandBase 里，因为只对 prompt 型有意义）：

- `source` — 信任域（`SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'`）
- `pluginInfo` — 如果来自 plugin，关联 manifest

**辅助**：

- `progressMessage` — 加载进度文案
- `contentLength` — 内容字符长度（token 估算）
- `argNames` — 参数名列表
- `disableNonInteractive` — 非交互模式禁用

#### `source` 跟 `loadedFrom` 是不同维度

`source` 在 PromptCommand（不在 CommandBase）里——只有 prompt 型有 source 信任域概念。

| `source` 取值 | 信任级别 | 典型场景 |
|---|---|---|
| `policySettings` | 最高（admin trusted） | 企业策略下发 |
| `bundled` | admin trusted | CLI 自带 |
| `builtin` | admin trusted | 内建命令（不参与 SkillTool） |
| `userSettings` | 用户级 | `~/.claude/skills/` |
| `projectSettings` | 项目级 | `<repo>/.claude/skills/` |
| `localSettings` | 项目本地（gitignored） | (skill 不用) |
| `flagSettings` | CLI flag | (skill 不用) |
| `plugin` | 取决于 plugin 来源 | 第三方 plugin |
| `mcp` | 远程不可信 | MCP server |

**`source` 决定权限策略**——典型用法：`isSourceAdminTrusted(source)` 判断是否能注册 hooks（`processSlashCommand.tsx:870`）。

`loadedFrom` 决定 UI 分组和过滤——`/skills` 菜单按它分组。

为什么不能合并？04 篇 6 节那张矩阵讲过：同一个文件可以 `source: projectSettings, loadedFrom: skills`，也可以 `source: projectSettings, loadedFrom: commands_DEPRECATED`——`source` 看"配置存在哪一层"，`loadedFrom` 看"通过哪条 loader 路径进来"，两者**正交**。

### 4.2 LocalCommand（74-78 行）— 跑本地代码、返回文本

```ts
type LocalCommand = {
  type: 'local'
  supportsNonInteractive: boolean
  load: () => Promise<LocalCommandModule>
}
```

入口签名（62-65）：

```ts
export type LocalCommandCall = (
  args: string,
  context: LocalJSXCommandContext,
) => Promise<LocalCommandResult>
```

返回值（16-23）：

```ts
export type LocalCommandResult =
  | { type: 'text'; value: string }
  | { type: 'compact'; compactionResult: CompactionResult; displayText?: string }
  | { type: 'skip' }
```

#### 这是什么

**本地命令——跑一段代码、产出一段文本**，不展开 prompt。典型例子：

- `/clear` — 清屏
- `/cost` — 显示当前成本
- `/help` — 显示帮助
- `/compact` — 触发压缩

注意 `LocalCommandResult` 三种情况：

- `text` — 普通文本结果，作为 user message 注入
- `compact` — 特殊：返回压缩结果（`/compact` 用）
- `skip` — 不做任何事（命令"取消"了）

**`load` 是 lazy import**——避免启动期把所有命令的实现代码都加载。

### 4.3 LocalJSXCommand（144-152 行）— 渲染 React UI

```ts
type LocalJSXCommand = {
  type: 'local-jsx'
  load: () => Promise<LocalJSXCommandModule>
}
```

入口签名（131-135）：

```ts
export type LocalJSXCommandCall = (
  onDone: LocalJSXCommandOnDone,
  context: ToolUseContext & LocalJSXCommandContext,
  args: string,
) => Promise<React.ReactNode>
```

#### 这是什么

**渲染交互 UI 的命令**。例子：

- `/skills` — 弹出 skill 列表菜单
- `/login` — 登录流程对话
- `/permissions` — 权限管理面板
- `/config` — 配置编辑器

签名上的关键差异：

- 返回 `React.ReactNode`——直接渲染
- 第一个参数 `onDone(result?, options?)`——异步回调，让 UI 通知"我做完了，结果是 X"
- `context` 多了 `LocalJSXCommandContext`（有 `setMessages`、`onChangeAPIKey` 等只在 UI 里用得到的字段）

`onDone` 的设计很有意思——React 的 imperative API 很不优雅，所以 LocalJSXCommand 用 callback 而不是返回 promise。这是个**"形态决定接口"**的好例子：UI 命令本质是用户驱动（用户点按钮），不是函数返回。

#### LocalJSXCommandContext（80-98）

```ts
export type LocalJSXCommandContext = ToolUseContext & {
  canUseTool?: CanUseToolFn
  setMessages: (updater: (prev: Message[]) => Message[]) => void
  options: { dynamicMcpConfig?, ideInstallationStatus?, theme }
  onChangeAPIKey: () => void
  onChangeDynamicMcpConfig?: ...
  onInstallIDEExtension?: ...
  resume?: (sessionId, log, entrypoint) => Promise<void>
}
```

这是 UI 命令独需的能力：改消息列表、切 API key、装 IDE 扩展、resume 历史会话。这些都是 UI 操作（用户点的按钮触发），不是 prompt 命令或本地代码命令该做的。

### 4.4 三类形态的对照

| 维度 | `prompt` | `local` | `local-jsx` |
|---|---|---|---|
| 执行结果 | prompt block 注入会话 | 文本 / compact / skip | React 节点 |
| 是否走主 query | **是**（注入后让模型继续） | **否**（结果就是结果） | **否**（UI 自己处理） |
| 异步模型 | `Promise<ContentBlock[]>` | `Promise<Result>` | `onDone` callback + `Promise<ReactNode>` |
| 谁跑 | 模型 | 本地代码 | React UI |
| 典型 | skill / `/init` | `/clear` | `/skills` 菜单 |

**这个三分法暴露了 Claude Code 的一个核心设计思想**——它把"命令"建模成**执行模型不同**的几族。不强行把所有命令塞进同一签名，避免"什么都能做但每件事都做得很挫"的过度抽象。

---

## 5. 三个独立 helper 函数

文件末尾还有几个工具函数：

### 5.1 `getCommandName`（209-211）

```ts
export function getCommandName(cmd: CommandBase): string {
  return cmd.userFacingName?.() ?? cmd.name
}
```

8 行解决 plugin 命名问题。`findCommand`（commands.ts）的查找逻辑是：

```ts
return commands.find(_ =>
  _.name === commandName ||
  getCommandName(_) === commandName ||      // ← 这里
  _.aliases?.includes(commandName),
)
```

**用户敲 `/review` 能命中 plugin skill `my-plugin:review`** 就靠这条——内部 name 不变（避免冲名），用户视角名字短。

### 5.2 `isCommandEnabled`（214-216）

```ts
export function isCommandEnabled(cmd: CommandBase): boolean {
  return cmd.isEnabled?.() ?? true
}
```

3 行的封装。意义：**默认 true**（不写 `isEnabled` 等同于"启用"），调用方不用每次都 `cmd.isEnabled?.() ?? true`。

`commands.ts:484` 的过滤就是 `... && isCommandEnabled(_)`。

### 5.3 LocalJSXCommandOnDone（117-126）

```ts
export type LocalJSXCommandOnDone = (
  result?: string,
  options?: {
    display?: CommandResultDisplay     // 'skip' | 'system' | 'user'
    shouldQuery?: boolean
    metaMessages?: string[]
    nextInput?: string
    submitNextInput?: boolean
  },
) => void
```

LocalJSXCommand 的"完成回调"协议。每个字段都对应一种完成方式：

- `display: 'skip'` → 不在 transcript 留痕（用户取消对话）
- `shouldQuery: true` → 让主 query 继续（UI 命令也能触发模型）
- `metaMessages` → 注入 isMeta:true 的消息（隐藏给用户但模型能看到）
- `nextInput` / `submitNextInput` → UI 关闭后预填输入框 / 自动提交

这个 protocol 反映了 UI 命令的**完成态可能很多样**——不是所有命令完成都"产生一段文本"。

---

## 6. Command 在系统里的关系图

把 Command 放回 8 个 model 的关系网里：

```
                    ┌─────────────────────────┐
                    │  Command (this file)    │
                    │  (统一抽象)              │
                    └────────┬────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        PromptCommand   LocalCommand   LocalJSXCommand
              │
              │ 执行时调用 getPromptForCommand
              │ 返回 ContentBlockParam[]
              │ 这是 SDK Message 的 content
              │
              ▼
       ┌──────────────────────────────────┐
       │  Message (utils/messages.ts)     │
       │  user message 含 isMeta:true     │
       │  注入主 conversation             │
       └──────────────────────────────────┘
              │
              │ messages 由主 query 发给模型
              │
              ▼
       ┌──────────────────────────────────┐
       │  ToolUseContext (Tool.ts)        │
       │  包含 commands: Command[]        │
       │  执行 SkillTool 时拿命令表       │
       └──────────────────────────────────┘
              │
              ▼
       ┌──────────────────────────────────┐
       │  AppState (AppStateStore.ts)    │
       │  mcp.commands / plugins.commands │
       │  也是 Command[]                  │
       └──────────────────────────────────┘
              │
              │ Command 的 source 决定
              ▼
       ┌──────────────────────────────────┐
       │  Permission (types/permissions)  │
       │  isSourceAdminTrusted(source)    │
       │  决定 hooks 注册策略             │
       └──────────────────────────────────┘
              │
              │ Command 的 hooks 字段
              ▼
       ┌──────────────────────────────────┐
       │  Hook (types/hooks.ts)           │
       │  registerSkillHooks 时注册       │
       └──────────────────────────────────┘
              │
              │ Command 来自 plugin 时
              ▼
       ┌──────────────────────────────────┐
       │  Plugin (types/plugin.ts)        │
       │  PromptCommand.pluginInfo 关联   │
       │  manifest                        │
       └──────────────────────────────────┘
```

**Command 是 8 个 model 里"扇出"最多的**——它直接或间接关联 Message、Tool、AppState、Permission、Hook、Plugin。这正是它叫"统一抽象"的原因——它是命令系统的**总线**。

---

## 7. 这一篇你应该带走的几样东西

读到这里，你应该能：

1. 解释 `Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)` 的含义和必要性
2. 区分 5 个可见性字段（`disableModelInvocation` / `userInvocable` / `isHidden` / `isEnabled` / `availability`）各自决定什么、为什么不能合并
3. 解释 `name` / `userFacingName` / `aliases` 三者的分工
4. 解释 `description` / `whenToUse` / `hasUserSpecifiedDescription` 的设计意图
5. 解释 `source` 和 `loadedFrom` 是正交的两个维度，各自驱动什么决策
6. 区分三类执行形态（prompt / local / local-jsx）在执行模型上的根本差异
7. 解释 `LocalJSXCommandOnDone` 的 callback 设计为什么不是 promise

---

## 8. 设计哲学要点（提炼到 08 篇会再展开）

这一个 model 暴露了 Claude Code 几个明确的设计选择：

1. **类型驱动的多态**：用 discriminated union（`type` 字段）替代继承——简单，TS 友好。
2. **"形态决定接口"**：执行模型不同的命令用不同签名，不强行抽象。
3. **静态描述 + 动态修饰**：CommandBase 主要静态，但 `isEnabled` 是函数（动态），`hooks` 在调用时才注册（运行时副作用）。
4. **来源是一等公民**：`source` 和 `loadedFrom` 都被建模——信任域和装载路径都是会影响行为的元数据。
5. **可见性正交于行为**：`isHidden`（UI）和 `userInvocable`（行为）允许独立——避免"UI 决定行为"或"行为决定 UI"的耦合。
6. **跨受众分字段**：`description`（人）/ `whenToUse`（模型）显式分开。
7. **运行时上下文修饰**：PromptCommand 通过 `allowedTools` / `model` / `effort` / `context` / `hooks` 临时改本轮执行——命令不只是内容载体，是"执行策略包"。

下一篇 02 看 `Tool`——它和 Command 是另一对核心抽象。
