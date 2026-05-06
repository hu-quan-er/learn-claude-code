# 04. 使用与执行机制

## 1. 两条主执行链

skill 的执行在项目里分成两条主链：

### 1.1 用户显式调用

用户输入：

```text
/commit
/review-pr 123
/my-skill foo bar
```

走的是：

- `restored-src/src/utils/processUserInput/processSlashCommand.tsx`

### 1.2 模型主动调用

模型通过工具调用：

```json
{ "skill": "commit", "args": "..." }
```

走的是：

- `restored-src/src/tools/SkillTool/SkillTool.ts`

这两条链最终都会复用 skill 的 `getPromptForCommand()`，但权限、日志、消息包装和上下文修改方式不一样。

## 2. 用户 `/skill` 的执行链

### 2.1 解析

`processSlashCommand()` 先做：

1. `parseSlashCommand(inputString)`
2. 从 `context.options.commands` 里查 `findCommand()`
3. 判断是否真的是命令
4. 如果是 `prompt` 类型，则进入 `getMessagesForPromptSlashCommand()`

### 2.2 `prompt` 型 skill 的关键动作

`getMessagesForPromptSlashCommand()` 会做这些事：

1. 调 `command.getPromptForCommand(args, context)` 拿到展开后的内容
2. 如果该 skill 声明了 `hooks`，尝试注册 hooks
3. 调 `addInvokedSkill()` 把 skill 内容存起来，给 compact 用
4. 生成一条 command loading metadata
5. 从 `allowedTools` 生成 `command_permissions` attachment
6. 把 skill 内容作为 `isMeta: true` 的 user message 注入上下文
7. 返回 `shouldQuery: true`，让主会话继续 query

这说明用户执行 `/skill` 不等于“本地命令执行完成”。

真正的效果是：

- 向当前回合注入一段 skill prompt
- 然后让模型继续看着这段 prompt 工作

## 3. 为什么会有 metadata message

skill 被加载时会生成类似：

- `<command-message>...`
- `<command-name>...</command-name>`
- `<skill-format>true</skill-format>`

这些 tag 的作用不是展示给用户，而是给消息渲染、工具系统和重复调用保护用。

尤其 `SkillTool/prompt.ts` 里明确写了：

- 如果当前 turn 里已经出现 `<COMMAND_NAME_TAG>`
- 说明 skill 已经加载
- 模型不要重复调用 SkillTool

也就是说，skill 调用不是幂等文本展开，而是带有状态标记的上下文注入。

## 4. `inline` 和 `fork` 的执行差异

### 4.1 inline

默认模式。

行为是：

- 直接把 skill prompt 注入主会话
- 主 agent 在当前上下文里继续执行

适合：

- 需要用户中途干预
- 需要沿用当前会话上下文
- 工作量不值得单开子 agent

### 4.2 fork

如果 skill 声明：

```yaml
context: fork
agent: xxx
```

那么不走 inline，而是：

- `prepareForkedCommandContext()`
- `runAgent()`

用户 slash 命令这边走：

- `executeForkedSlashCommand()`

模型 SkillTool 这边走：

- `executeForkedSkill()`

fork 模式下，skill 不再只是 prompt 注入，而是真正变成一个子 agent 任务。

## 5. 模型 `SkillTool` 的执行链

`SkillTool.ts` 的职责比 slash command 更复杂，因为它要负责：

- 校验 skill 是否存在
- 判断模型是否允许调用
- 权限决策
- fork 或 inline 分流
- 把 skill 调用结果转换成 tool_result
- 把 skill 带来的上下文修改传回主执行器

## 6. `SkillTool.validateInput()` 做了什么

主要检查：

1. skill 名是否合法
2. 去掉可能存在的前导 `/`
3. skill 是否存在于可用命令集合
4. 是否 `disableModelInvocation`
5. 是否真的是 `type === 'prompt'`

如果这些检查失败，模型就不能通过 SkillTool 硬猜一个不存在的 skill。

## 7. `SkillTool` 的权限机制

这部分很值得单独看。

### 7.1 deny/allow 规则

`checkPermissions()` 会先看用户配置的：

- deny rules
- allow rules

并支持：

- 精确匹配
- `prefix:*` 前缀匹配

### 7.2 safe skill 自动放行

如果 skill 只包含一组“安全属性”，系统会自动允许，不弹权限确认。

这个判断通过：

- `SAFE_SKILL_PROPERTIES`
- `skillHasOnlySafeProperties()`

来完成。

换句话说，SkillTool 并不是“所有 skill 都要用户确认”。

它把 skill 本身也做了风险分级。

### 7.3 非安全 skill 会要求用户授权

默认 ask 时还会生成两种建议规则：

- 只允许这个 skill
- 允许这个 skill 的前缀模式

说明 skill 权限是正式纳入 permission system 的，而不是额外 hack。

## 8. 模型触发 inline skill 时，如何真正复用 slash command 流程

这点设计很漂亮。

`SkillTool.call()` 在 inline 情况下并没有重写一套 skill 注入逻辑，而是直接：

- 动态 import `processPromptSlashCommand()`
- 用它处理 skill

也就是说：

- 用户 `/commit`
- 模型 `SkillTool(skill="commit")`

最终都会落到同一条 prompt 展开逻辑上。

这样做的好处是：

- hooks 注册不会出现两套逻辑
- compact 保留逻辑不会分叉
- metadata tag 格式一致
- 附加权限 attachment 一致

## 9. SkillTool 如何把 skill 的副作用传回主上下文

这部分是 `contextModifier()`。

当 skill 处理完成后，SkillTool 不只是返回“启动了 skill”，还会把 skill 运行带来的环境修改注回去。

主要有三类：

### 9.1 `allowedTools`

把 skill 的 `allowedTools` 合并进：

- `toolPermissionContext.alwaysAllowRules.command`

效果是：

- 本轮后续执行可以直接用这些工具

### 9.2 `model`

如果 skill 指定了模型，会用：

- `resolveSkillModelOverride()`

去覆盖当前主循环模型。

### 9.3 `effort`

如果 skill 指定了 effort，会把当前 app state 的 `effortValue` 覆盖掉。

所以 skill 不只是“加一段提示词”，它还能临时改当前回合的执行配置。

## 10. hook 注册机制

skill 的 hooks 注册发生在：

- `getMessagesForPromptSlashCommand()`

逻辑是：

1. 只有 skill 被真正加载时才注册
2. 还要检查 `hooks` 是否允许在当前策略下启用
3. `registerSkillHooks()` 会把 hook 绑定到当前 session

这里还有一条策略约束：

- 如果系统是 plugin-only / 受限模式
- 只有被认为 admin-trusted 的来源才能注册 hook

也就是说，skill 可以扩展行为，但不能绕过策略面。

## 11. compact 后为什么 skill 不会丢

skill 一旦被调用，`processSlashCommand.tsx` 会执行：

- `addInvokedSkill(command.name, skillPath, skillContent, agentId)`

之后在 compact 过程中：

- `services/compact/compact.ts`
- `createSkillAttachmentIfNeeded()`

会把最近调用过的 skill 内容塞进 `invoked_skills` attachment。

这解决的是一个很实际的问题：

- conversation 被总结压缩后
- 原本已经展开进上下文的 skill 内容会丢
- 如果不保留，模型之后会忘记这个 skill 的规则

所以 invoked skill tracking 是这套机制的关键补丁，而不是锦上添花。

## 12. agent 维度隔离

invoked skills 不是全局一锅端保存，而是按 `agentId` 分组。

这意味着：

- 主会话调用的 skill 只回到主会话
- fork 子 agent 调用的 skill 只保留给那个 agent

这样做是为了防止跨 agent 泄漏：

- A agent 用的 skill 不应该在 B agent compact 后继续生效

这是 `bootstrap/state.ts` 里 `invokedSkills` 设计成复合 key 的原因。

## 13. skill 的“被动提示”机制

除了显式调用，skill 还有两种被动进入模型视野的方式。

### 13.1 `skill_listing` attachment

`utils/attachments.ts` 会把尚未发过的 skill 名单附给模型。

特点：

- 按预算裁剪描述
- 按 agent 记录已发送名字
- 动态新增 skill 会继续补发

### 13.2 skill discovery guidance

`constants/prompts.ts` 会告诉模型：

- 当前 relevant skills 会自动出现
- 如果当前列表不够，再调用 DiscoverSkills

这让 skill 使用不只是“模型记得某个 skill 名字”，而是“系统持续提醒模型有哪些 skill 可用”。

## 14. `disableModelInvocation` 和 `userInvocable` 的实际效果

这两个字段常被混淆。

### `userInvocable: false`

效果：

- 用户不能直接 `/skill-name`
- 但模型仍可通过 SkillTool 调用

用户会看到一条说明：

- 这个 skill 只能由 Claude 调用

### `disableModelInvocation: true`

效果：

- 模型不能通过 SkillTool 调用
- 但用户仍可能能直接 `/skill-name`

典型例子是 bundled `skillify`：

- 用户可以显式发起
- 模型不能自己随便触发

## 15. skill 调用的 telemetry

无论是 slash 命令还是 SkillTool，skill 调用都会记录 telemetry。

记录内容包括：

- skill 名
- source
- loadedFrom
- kind
- plugin 信息
- execution_context 是 `inline` 还是 `fork`
- query depth
- was_discovered

这说明 skill 不是“提示词黑箱”，而是产品强观测能力。

## 16. 这一层的本质

skill 的“使用”本质上是一次受控的执行环境切换：

- 注入专门 prompt
- 修改本轮权限
- 可能切模型
- 可能切 effort
- 可能注册 hooks
- 可能切到子 agent
- 可能在 compact 后继续保留

所以 skill 不是普通 slash command 的别名。

它更像一个“临时加载的执行策略包”。
