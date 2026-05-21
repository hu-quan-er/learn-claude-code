# 05. Permission — 权限规则与决策模型

> 这一篇做一件事：把 `restored-src/src/types/permissions.ts` 全部 441 行**作为 model**讲透——permission mode、rule、decision、result、reason、auto classifier、explainer 一整套类型体系。读完之后你能解释"为什么权限决策需要 9 种 reason / 4 种 behavior / 5 种 mode 的组合"。
>
> 接 02 篇：Tool.checkPermissions 返回 `PermissionResult`；ToolUseContext 含 `ToolPermissionContext`。本篇是这套类型的 model 视角。

---

## 1. Permission 在所有 model 里的位置

Claude Code 是一个**会跑外部命令、改本地文件、调远程 API** 的 agent CLI。"哪些操作要先问用户"是它的核心安全边界。

权限模型解决的问题：

- 用户和管理员**通过规则**（settings.json / managed-settings.json）声明哪些自动放行 / 拒绝 / 询问
- 模型 / tool **触发某个操作**时，权限系统按规则决策
- 决策结果可能是 allow / deny / ask / passthrough
- 决策**附带原因**（哪条规则、哪个 hook、哪个 classifier 决定的）
- 决策可以**动态调整 input**（比如改文件路径、修改 args）
- 用户在 ask 对话框里可以**保存新规则**（"以后总是允许"）

这一切都通过 `types/permissions.ts` 的 8 大类型表达。

---

## 2. 文件结构

打开 `restored-src/src/types/permissions.ts`，按注释分了 8 段：

```
permissions.ts (1-441)
│
├── 段 0  imports                              1-10
│
├── 段 1  Permission Modes                    13-39      ★ 5 种模式
│
├── 段 2  Permission Behaviors                41-44      4 种 behavior
│
├── 段 3  Permission Rules                    47-79      ★ rule 三元组
│
├── 段 4  Permission Updates                  82-146     6 种更新操作
│
├── 段 5  Permission Decisions & Results     149-266    ★ 决策结果联合
│
├── 段 6  Permission Decision Reasons        268-324    9 种原因
│
├── 段 7  Bash Classifier Types              326-397    YOLO 分类器
│
├── 段 8  Permission Explainer Types         399-410
│
└── 段 9  Tool Permission Context             412-441    ★ runtime 字段
```

---

## 3. 段 1：Permission Modes（13-39）

```ts
export const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',
  'bypassPermissions',
  'default',
  'dontAsk',
  'plan',
] as const

export type ExternalPermissionMode = (typeof EXTERNAL_PERMISSION_MODES)[number]
export type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
export type PermissionMode = InternalPermissionMode

export const INTERNAL_PERMISSION_MODES = [
  ...EXTERNAL_PERMISSION_MODES,
  ...(feature('TRANSCRIPT_CLASSIFIER') ? (['auto'] as const) : []),
] as const satisfies readonly PermissionMode[]
```

### 3.1 模式语义

| Mode | 行为 |
|---|---|
| `default` | 标准 — 按规则决策，需要用户确认 |
| `acceptEdits` | 文件编辑自动允许（仍要确认 destructive） |
| `bypassPermissions` | 全部自动允许（"YOLO" 模式） |
| `dontAsk` | 不弹任何对话框（headless / 后台） |
| `plan` | 计划模式 — 模型只能读不能写 |
| `auto` | 实验性 — 用 classifier 自动决策（gated by feature flag） |
| `bubble` | 内部 — 把权限问题"冒泡"到调用方处理 |

### 3.2 `External` vs `Internal` 的拆分

这两组的差别很微妙：

```ts
ExternalPermissionMode = 'acceptEdits' | 'bypassPermissions' | 'default' | 'dontAsk' | 'plan'
InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble'
```

为什么要分？

- **External** 是**用户能在 settings.json / CLI flag 里写的值**——是面向用户的稳定 API
- **Internal** 多了 `'auto'`（实验性，feature flag 控制）和 `'bubble'`（内部只用）——这两个用户不该直接配置

`PermissionMode` 实际等于 `InternalPermissionMode`——所有 runtime 类型都接受 7 种值。但 settings / CLI 只校验前 5 种。

### 3.3 `INTERNAL_PERMISSION_MODES` 的运行时数组

注释 31-32：

```
Runtime validation set: modes that are user-addressable (settings.json
defaultMode, --permission-mode CLI flag, conversation recovery).
```

**类型 + runtime 数组双重维护**：类型给 TS 编译期，数组给运行时校验（"用户传进来的字符串是不是合法值"）。`feature('TRANSCRIPT_CLASSIFIER')` 守卫让 `'auto'` 在功能开启时**才**进数组——动态调整可接受值集合。

---

## 4. 段 2：Permission Behaviors（41-44）

```ts
export type PermissionBehavior = 'allow' | 'deny' | 'ask'
```

**Rule 的取值**——一条 permission rule 可以是 allow、deny 或 ask 三选一。

注意这里**没有 `passthrough`**——passthrough 只出现在 `PermissionResult`（运行时决策结果），不能写在静态 rule 里。后面会展开。

---

## 5. 段 3：Permission Rules（47-79）— 规则三元组

### 5.1 类型定义

```ts
export type PermissionRuleSource =
  | 'userSettings'            // ~/.claude/settings.json
  | 'projectSettings'         // <repo>/.claude/settings.json
  | 'localSettings'           // <repo>/.claude/settings.local.json (gitignored)
  | 'flagSettings'            // CLI --settings flag
  | 'policySettings'          // managed-settings.json (admin)
  | 'cliArg'                  // CLI flag (--allowedTools / --deniedTools)
  | 'command'                 // 命令产生（skill 的 allowedTools）
  | 'session'                 // 用户在会话中"保存为规则"

export type PermissionRuleValue = {
  toolName: string
  ruleContent?: string
}

export type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: PermissionBehavior
  ruleValue: PermissionRuleValue
}
```

### 5.2 三元组：source × behavior × value

每条 rule 由 3 个独立维度构成：

```
(来源 source)        (行为 behavior)       (具体规则 value)
userSettings          allow                  Bash(git:*)
projectSettings       deny                   Read
session               ask                    Skill(verify)
cliArg                allow                  WebFetch
```

#### `source` 8 种来源

注释 51-53：

```
Where a permission rule originated from.
Includes all SettingSource values plus additional rule-specific sources.
```

为什么需要分这么细？

- **优先级**：政策（policySettings）覆盖用户（userSettings）覆盖项目（projectSettings）覆盖本地（localSettings）
- **持久化**：`session` 不持久（关 Claude Code 就消失），其它落盘
- **可见性**：UI 展示规则来自哪——让用户知道"这条规则是我配的还是 IT 配的"
- **冲突解决**：多源都对同一 tool 有规则时，按 source 优先级合并

8 个 source 的拆分让上面 4 个能力都能精确实现。

#### `ruleValue` 的 toolName + ruleContent

两个字段：

- `toolName: string` — 工具名（Bash / Read / Skill / mcp__github__create_issue 等）
- `ruleContent?: string` — 可选的细化匹配（"git:*" / 具体路径 / skill 名前缀）

例子：

- `Bash(git status:*)` → toolName='Bash', ruleContent='git status:*'
- `Read` → toolName='Read', ruleContent=undefined（全部 Read）
- `Skill(review:*)` → toolName='Skill', ruleContent='review:*'

`ruleContent` 的语法**由各 tool 的 `preparePermissionMatcher` 解释**（Tool.ts:514-516）——不同 tool 有不同的匹配规则。Bash 是 glob 风格，Read/Edit 是路径风格，Skill 是名字前缀。

#### 思考题

> 如果 `userSettings` 写 `allow Bash(git:*)`，但 `localSettings` 写 `deny Bash(git push:*)`——用户敲 `git push` 时被允许还是拒绝？为什么？

（提示：deny 通常优先于 allow，无论 source。但 source 优先级也起作用——具体由权限引擎的合并策略决定。最常见的实现是 deny-wins。）

---

## 6. 段 4：Permission Updates（82-146）— 6 种更新操作

```ts
export type PermissionUpdate =
  | { type: 'addRules'; destination, rules, behavior }
  | { type: 'replaceRules'; destination, rules, behavior }
  | { type: 'removeRules'; destination, rules, behavior }
  | { type: 'setMode'; destination, mode }
  | { type: 'addDirectories'; destination, directories }
  | { type: 'removeDirectories'; destination, directories }
```

### 6.1 6 种 mutation

按 `type` 判别。这是**修改权限配置**的标准操作集：

- 增删改 rules：3 种（add / replace / remove）
- 改 mode：1 种
- 增删 working directory：2 种

### 6.2 `PermissionUpdateDestination` (88-93)

```ts
export type PermissionUpdateDestination =
  | 'userSettings'
  | 'projectSettings'
  | 'localSettings'
  | 'session'
  | 'cliArg'
```

注意**比 `PermissionRuleSource` 少 3 个**（policySettings、command、session 之一）：

- `policySettings` 是 admin 控制——用户的 update 不能改它
- `command` 是命令运行时产生的——不能 destination 到这（命令再次跑会重新生成）

剩下 5 个是用户能"保存到"的目的地。

### 6.3 用例：用户在权限对话框里点"始终允许"

权限对话框给用户两个选项："just this time" / "always"。"always" 触发：

```ts
{
  type: 'addRules',
  destination: 'localSettings',  // 默认本地（不污染 git 仓库）
  rules: [{ toolName: 'Bash', ruleContent: 'git status:*' }],
  behavior: 'allow'
}
```

这就是"建议规则"在 06 篇 5.3 节看到的 SkillTool `checkPermissions` 返回的 `suggestions: PermissionUpdate[]`。

---

## 7. 段 5：Permission Decisions & Results（149-266）— 决策结果联合

这一段是 model **最微妙的部分**。

### 7.1 类型层次

```
PermissionResult
  = PermissionDecision | PassthroughResult
                ↓
  PermissionDecision = Allow | Ask | Deny

  Allow:        { behavior: 'allow', updatedInput?, ... }
  Ask:          { behavior: 'ask', message, suggestions?, ... }
  Deny:         { behavior: 'deny', message, decisionReason }

  Passthrough:  { behavior: 'passthrough', message, ... }
```

`PermissionDecision` 是"具体决定"，`PermissionResult` 是"决定 + 透传"——多了一个 `'passthrough'` 选项。

### 7.2 Allow Decision（173-184）

```ts
export type PermissionAllowDecision<Input> = {
  behavior: 'allow'
  updatedInput?: Input
  userModified?: boolean
  decisionReason?: PermissionDecisionReason
  toolUseID?: string
  acceptFeedback?: string
  contentBlocks?: ContentBlockParam[]
}
```

#### 几个值得看的字段

- `updatedInput` — 决策可能**修改 input**（最典型：路径绝对化、参数补齐）
- `userModified` — 用户在对话框里手动改过 input（影响 telemetry）
- `decisionReason` — 为什么 allow（哪条规则）
- `acceptFeedback` — 用户接受时附加的反馈文本
- `contentBlocks` — 用户附加的内容（粘贴的图片）

**"决策不只是 yes/no"**——它还能改 input、附图、附反馈。这种丰富建模反映了真实交互场景。

### 7.3 Ask Decision（199-226）

```ts
export type PermissionAskDecision<Input> = {
  behavior: 'ask'
  message: string
  updatedInput?: Input
  decisionReason?: PermissionDecisionReason
  suggestions?: PermissionUpdate[]
  blockedPath?: string
  metadata?: PermissionMetadata
  isBashSecurityCheckForMisparsing?: boolean
  pendingClassifierCheck?: PendingClassifierCheck
  contentBlocks?: ContentBlockParam[]
}
```

#### 关键字段

- `message` — 给用户看的提问
- `suggestions: PermissionUpdate[]` — "始终允许"等按钮对应的 update（点击就保存）
- `blockedPath` — 涉及具体路径时，UI 用它聚焦展示
- `metadata` — 附带 command 元数据（让 UI 展示来源、描述等）
- `pendingClassifierCheck` — 后台跑 classifier 的 metadata，可能在用户回答前自动 allow

#### `pendingClassifierCheck` 是什么

注释 187-194：

```
Metadata for a pending classifier check that will run asynchronously.
Used to enable non-blocking allow classifier evaluation.
```

```ts
export type PendingClassifierCheck = {
  command: string
  cwd: string
  descriptions: string[]
}
```

**异步分类器并行**：用户看到对话框时，后台 LLM 分类器同时跑。如果分类器**早于**用户回答完成 + 判断"安全"，对话框直接 auto-allow。这是 latency 优化——大多数命令是安全的，让用户少按一次按钮。

#### `isBashSecurityCheckForMisparsing` 的具体场景

注释 209-214 描述：

```
If true, this ask decision was triggered by a bashCommandIsSafe_DEPRECATED security check
for patterns that splitCommand_DEPRECATED could misparse (e.g. line continuations, shell-quote
transformations). Used by bashToolHasPermission to block early before splitCommand_DEPRECATED
transforms the command. Not set for simple newline compound commands.
```

这是**对 bash 命令解析的不信任**——`splitCommand_DEPRECATED` 函数不能正确处理某些 shell 语法（行继续 `\`、shell-quote 转换）。这种情况 ask 而不是 allow——保守。

字段名带 `_DEPRECATED` 暗示这部分代码已被新版替代，但仍要保留（兼容）。

### 7.4 Deny Decision（231-236）

```ts
export type PermissionDenyDecision = {
  behavior: 'deny'
  message: string
  decisionReason: PermissionDecisionReason       // 注意：必填
  toolUseID?: string
}
```

**Deny 的 `decisionReason` 必填**（非 optional）。原因：拒绝必须告诉用户"为什么"——"被规则拒绝"vs"被 hook 拒绝"vs"被 mode 拒绝"。

而 allow / ask 的 reason 是 optional，因为有时只是 default 流程。

### 7.5 PermissionResult 多一个 passthrough（249-266）

```ts
export type PermissionResult<Input> =
  | PermissionDecision<Input>
  | {
      behavior: 'passthrough'
      message: string
      decisionReason?: ...
      suggestions?: PermissionUpdate[]
      blockedPath?: string
      pendingClassifierCheck?: PendingClassifierCheck
    }
```

**`passthrough` 的语义**：tool 自己的 `checkPermissions` 不知道怎么决定，把决策**让给上层**（默认权限引擎）。

`Tool.ts:762-766` 的默认 `checkPermissions` 返回 `{ behavior: 'allow', updatedInput }`——直接放行。但有些 tool 会返回 passthrough，让通用规则引擎接管。

为什么不直接返回 allow？因为 tool 的"我不知道"和"我说允许"语义不同——前者是**未介入**，后者是**主动允许**。passthrough 让区分能保留。

---

## 8. 段 6：Permission Decision Reasons（268-324）— 9 种原因

```ts
export type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'other'; reason: string }
```

### 8.1 11 种 reason（不是 9 种——文件注释说 9 种但实际更多）

每种 reason 代表**决策的来源**：

| `type` | 含义 |
|---|---|
| `rule` | 命中了某条 PermissionRule（含完整 rule 对象） |
| `mode` | mode 决定的（`bypassPermissions` 全 allow，`plan` 阻止写） |
| `subcommandResults` | Bash 复合命令——每个子命令的结果合并 |
| `permissionPromptTool` | 自定义 permission prompt tool 决定 |
| `hook` | PreToolUse hook 决定（含 hook 名和 source） |
| `asyncAgent` | 异步子 agent 不能弹对话框，按规则降级 |
| `sandboxOverride` | sandbox 强制许可（`--dangerously-disable-sandbox`） |
| `classifier` | LLM 分类器决定 |
| `workingDir` | 涉及 working directory 边界 |
| `safetyCheck` | safety check（敏感路径如 `.claude/`、`.git/`） |
| `other` | 其它 |

每个 reason 的 schema 都不同——这是 discriminated union，每种带自己的参数。

### 8.2 `safetyCheck` 的 `classifierApprovable`

注释 314-320：

```
When true, auto mode lets the classifier evaluate this instead of
forcing a prompt. True for sensitive-file paths (.claude/, .git/,
shell configs) — the classifier can see context and decide. False
for Windows path bypass attempts and cross-machine bridge messages.
```

**有些 safety check 不能让 classifier 接手**——比如 Windows 路径绕过尝试（`C:\..\..\` 这种）。这种是明确攻击模式，classifier 也不该 allow。

字段精确建模"这种 safety 是不是可以让 LLM 评估"——细到这个程度。

### 8.3 为什么 reason 这么细致？

两个原因：

**(a) 用户体验**：拒绝时告诉用户具体为什么。"Tool blocked by rule X" 比 "blocked" 友好太多。
**(b) Telemetry / 调试**：哪种 reason 触发最多？哪些场景下用户在 `ask` 时被 classifier 抢答？这些数据指导优化。

`type: 'other'` 是 catchall——但生产里几乎不应该见到（除非 reason 系统漏了某场景）。

---

## 9. 段 7：Bash Classifier Types（326-397）

YOLO 模式（"全自动"）的 classifier 相关类型。这是**实验性**的——`auto` mode 用 classifier 决定 Bash 命令是否安全。

```ts
export type ClassifierResult = {
  matches: boolean
  matchedDescription?: string
  confidence: 'high' | 'medium' | 'low'
  reason: string
}

export type ClassifierBehavior = 'deny' | 'ask' | 'allow'

export type ClassifierUsage = {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
}

export type YoloClassifierResult = {
  thinking?: string
  shouldBlock: boolean
  reason: string
  unavailable?: boolean
  transcriptTooLong?: boolean
  model: string
  usage?: ClassifierUsage
  durationMs?: number
  promptLengths?: { systemPrompt, toolCalls, userPrompts }
  errorDumpPath?: string
  stage?: 'fast' | 'thinking'
  stage1Usage?: ClassifierUsage
  stage1DurationMs?: number
  stage1RequestId?: string
  stage1MsgId?: string
  stage2Usage?: ClassifierUsage
  stage2DurationMs?: number
  stage2RequestId?: string
  stage2MsgId?: string
}
```

### 9.1 双阶段 classifier

`stage: 'fast' | 'thinking'` 反映**两阶段决策**：

1. **stage 1 fast** — 用快模型（haiku）快速过一遍。多数命令在这里就能判
2. **stage 2 thinking** — 不确定的进 stage 2，用 thinking 模式深入分析

每阶段的 usage、duration、requestId 全部记录——精细化分析"快路径命中率"、"深思考准确率"。

### 9.2 关键字段

- `transcriptTooLong` — API 返回 "prompt is too long"，意味 transcript 太长。**deterministic**（同一 transcript 一定再次失败），所以 caller 应该 fallback 到正常 prompt——不要 retry。
- `stage1RequestId` / `stage2RequestId` (`req_xxx`)——和服务端 api_usage 日志关联（性能分析）。
- `stage1MsgId` / `stage2MsgId` (`msg_xxx`)——和分类器决策的 actual prompt 关联。

这种**精细到追踪每个 classifier 调用的 trace ID**反映了 Claude Code 对自动决策系统的高度可观测性要求——黑盒决策必须可审计。

---

## 10. 段 8：Permission Explainer Types（399-410）

```ts
export type RiskLevel = 'LOW' | 'MEDIUM' | 'HIGH'

export type PermissionExplanation = {
  riskLevel: RiskLevel
  explanation: string
  reasoning: string
  risk: string
}
```

简短一段。Explainer 是**给用户解释"这个操作为什么有风险"**——比如 "Bash(rm -rf /)" 的 risk 解释。

`risk` vs `riskLevel`——前者是文字描述，后者是枚举（用于颜色 / 图标）。

`reasoning` 是分类器或 LLM 的思考过程——暴露给用户提升透明度。

---

## 11. 段 9：Tool Permission Context（412-441）

```ts
export type ToolPermissionRulesBySource = {
  [T in PermissionRuleSource]?: string[]
}

export type ToolPermissionContext = {
  readonly mode: PermissionMode
  readonly additionalWorkingDirectories: ReadonlyMap<string, AdditionalWorkingDirectory>
  readonly alwaysAllowRules: ToolPermissionRulesBySource
  readonly alwaysDenyRules: ToolPermissionRulesBySource
  readonly alwaysAskRules: ToolPermissionRulesBySource
  readonly isBypassPermissionsModeAvailable: boolean
  readonly strippedDangerousRules?: ToolPermissionRulesBySource
  readonly shouldAvoidPermissionPrompts?: boolean
  readonly awaitAutomatedChecksBeforeDialog?: boolean
  readonly prePlanMode?: PermissionMode
}
```

**这是运行时的"权限快照"**。Tool.ts 也定义了同名类型（`Tool.ts:123-138`）——`types/permissions.ts` 这一份是**类型版本**（去掉 DeepImmutable 包装），Tool.ts 加上了 DeepImmutable。

字段语义：

- `mode` — 当前 PermissionMode
- `additionalWorkingDirectories` — `--add-dir` 加的目录
- `alwaysAllowRules` / `alwaysDenyRules` / `alwaysAskRules` — 三类 rule 按 source 分组
- `isBypassPermissionsModeAvailable` — 这个 session 能不能用 `bypassPermissions`（policy 控制）
- `strippedDangerousRules` — 被自动移除的"过于宽松"规则
- `shouldAvoidPermissionPrompts` — 后台 agent 不能弹对话框
- `awaitAutomatedChecksBeforeDialog` — coordinator worker 等 classifier 完成再弹
- `prePlanMode` — 进 plan mode 之前是什么 mode（退出时恢复）

### 11.1 三类 rule + readonly Map 的设计

```ts
ToolPermissionRulesBySource = { [T in PermissionRuleSource]?: string[] }
```

按 source 分组的 string[]。比如：

```ts
alwaysAllowRules: {
  userSettings: ['Bash(git status:*)', 'Read'],
  projectSettings: ['Bash(npm test)'],
  session: ['Skill(verify)'],
}
```

按 source 分组的好处：**清空某 source 不影响其它**。重新装载 plugin 时清掉 `plugin` source 的所有 rules——一行 `delete rulesBySource.plugin`。

### 11.2 `additionalWorkingDirectories: ReadonlyMap`

为什么用 Map 不用对象？因为 path 可能含特殊字符（`-`、`.`），对象 key 都是 string 但操作没 Map 直观。`ReadonlyMap` 强制 immutable。

### 11.3 `prePlanMode` 的存在

```ts
prePlanMode?: PermissionMode
```

注释（Tool.ts:136-137）：

```
Stores the permission mode before model-initiated plan mode entry, so it can be restored on exit
```

**模型可以主动进入 plan mode**（通过 ExitPlanModeTool）——退出时要恢复原 mode。这字段做"栈"——push 当前 mode，pop 时恢复。

---

## 12. 整套权限决策流程

把 8 大类型串起来：

```
       配置阶段
       ┌────────────────────────┐
       │  settings.json         │
       │  / managed-settings    │
       │  / CLI flags           │
       └──────────┬─────────────┘
                  │ parse
                  ▼
       ┌────────────────────────┐
       │ ToolPermissionContext  │
       │ (在 AppState 里持久)    │
       │   mode                 │
       │   alwaysAllowRules     │
       │   alwaysDenyRules      │
       │   alwaysAskRules       │
       └──────────┬─────────────┘
                  │
                  │ tool 想跑时
                  ▼
       ┌────────────────────────┐
       │ tool.checkPermissions  │
       │ (Tool.ts 接口)          │
       └──────────┬─────────────┘
                  │ 返回
                  ▼
       ┌────────────────────────┐
       │ PermissionResult        │
       │ allow / deny / ask /   │
       │ passthrough            │
       │ + decisionReason        │
       └──────────┬─────────────┘
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
       allow     deny       ask
        │         │         │
        │         │         │ 弹对话框
        │         │         │ + suggestions: PermissionUpdate[]
        │         │         │
        │         │    ┌────▼─────────────────┐
        │         │    │ User clicks "always"  │
        │         │    └────┬─────────────────┘
        │         │         │ apply update
        │         │         ▼
        │         │    ┌────────────────────┐
        │         │    │ PermissionUpdate   │
        │         │    │ (addRules to       │
        │         │    │  localSettings)    │
        │         │    └────┬───────────────┘
        │         │         │
        │         │         ▼
        │         │    settings.local.json
        │         │    (next session 生效)
        │         │
        ▼         ▼
       tool.call  tool blocked
```

---

## 13. 这一篇你应该带走的几样东西

1. PermissionMode 7 种（External 5 + Internal 2 实验性 / 内部）
2. PermissionRule 三元组：source × behavior × value
3. 8 种 source 的优先级 / 持久化 / 可见性差异
4. PermissionResult 4 种 behavior（allow / deny / ask / passthrough），passthrough 仅出现在 result 不出现在 rule
5. Allow / Ask / Deny decision 各自的字段差异（reason 必填 vs 可选）
6. `pendingClassifierCheck` 是异步分类器并行优化
7. PermissionDecisionReason 11 种（rule / mode / hook / classifier / safetyCheck 等）
8. 双阶段 classifier（fast → thinking）的设计和字段
9. ToolPermissionContext 是运行时快照，按 source 分组让批量清理简单
10. `prePlanMode` 是 mode 栈式恢复

---

## 14. 设计哲学要点

1. **多维独立建模**：source / behavior / value / mode / reason 各自独立——不强行合并
2. **External vs Internal 公开 API 边界**：用户能写的值集和系统能用的值集分开
3. **决策不止 yes/no**：updatedInput / suggestions / contentBlocks 让决策能改 input、提建议、附图
4. **拒绝必须有理由**：Deny 的 decisionReason 必填——透明度优先
5. **passthrough 让 tool 弃权**：和 allow 语义不同——能显式表达"我不参与决策"
6. **异步并行优化**：`pendingClassifierCheck` 让对话框和 classifier 同时跑
7. **trace ID 全链路**：classifier 的 stage1RequestId / stage2MsgId 等让 LLM 决策可追溯
8. **mode 栈**：plan mode 进退用 prePlanMode 栈
9. **按 source 分组的 rules**：批量清理友好

下一篇 06 看 `Hook` —— Permission 的 reason 之一是 `hook`，hook 自身的 model 在那。
