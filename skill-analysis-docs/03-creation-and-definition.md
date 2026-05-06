# 03. 创建方式与定义格式

## 1. file-based skill 的标准形态

当前实现里，标准本地 skill 推荐形态是：

```text
.claude/skills/<skill-name>/SKILL.md
```

或者用户级：

```text
~/.claude/skills/<skill-name>/SKILL.md
```

`loadSkillsFromSkillsDir()` 明确只支持这种目录式格式。

它意味着：

- skill 名字天然取目录名
- skill 可以有自己的资源目录
- 后续可以通过 `CLAUDE_SKILL_DIR` 或 `Base directory for this skill` 找到旁边的脚本、模板、参考文档

## 2. `SKILL.md` 的 frontmatter 字段

核心解析逻辑在：

- `restored-src/src/utils/frontmatterParser.ts`
- `restored-src/src/skills/loadSkillsDir.ts`

### 常用字段

| 字段 | 作用 |
| --- | --- |
| `name` | 展示名，不改内部命令名 |
| `description` | 简短描述 |
| `when_to_use` | 告诉模型什么情况下应该用这个 skill |
| `allowed-tools` | skill 生效后临时追加允许的工具 |
| `arguments` | 参数名列表 |
| `argument-hint` | 参数提示 |
| `model` | skill 级模型覆盖 |
| `effort` | skill 级 effort 覆盖 |
| `user-invocable` | 用户是否可直接 `/skill-name` |
| `disable-model-invocation` | 模型是否可通过 `SkillTool` 调用 |
| `hooks` | 调用 skill 时注册的 hooks |
| `context` | `inline` 或 `fork` |
| `agent` | fork 时子 agent 类型 |
| `paths` | 条件激活路径规则 |
| `shell` | skill 内 `!` shell block 使用的 shell 类型 |

## 3. frontmatter 解析后的关键产物

`parseSkillFrontmatterFields()` 会把 frontmatter 变成一组标准运行时字段。

输出内容包括：

- `displayName`
- `description`
- `allowedTools`
- `argumentHint`
- `argumentNames`
- `whenToUse`
- `version`
- `model`
- `disableModelInvocation`
- `userInvocable`
- `hooks`
- `executionContext`
- `agent`
- `effort`
- `shell`

然后再交给 `createSkillCommand()` 统一生成 `Command` 对象。

## 4. `createSkillCommand()` 真正做了什么

`createSkillCommand()` 的重点不是“保存元数据”，而是构造 `getPromptForCommand()`。

它在运行时会做下面几步：

1. 如果有 `baseDir`，在 skill 内容前面注入：
   `Base directory for this skill: <dir>`
2. 用 `substituteArguments()` 做参数替换
3. 把 `${CLAUDE_SKILL_DIR}` 替换成 skill 自己的目录
4. 把 `${CLAUDE_SESSION_ID}` 替换成当前 session id
5. 对非 MCP skill 执行 `executeShellCommandsInPrompt()`
6. 最后返回 text block

这一层说明 skill 文件不是“原文直送模型”。

它是一个带有轻量模板能力和 shell 扩展能力的声明式 prompt 资源。

## 5. 参数替换机制

skill 定义里的参数主要靠：

- `arguments`
- `argument-hint`
- markdown 中的参数占位

来工作。

从实现上看，参数不是复杂 AST，而是基于 `substituteArguments()` 的文本替换。

因此写 skill 时更像是在写：

- 一段有命名参数插槽的 prompt 模板

而不是定义强类型函数签名。

## 6. shell block 能力

skill 支持在 markdown 里嵌入 shell 执行块，相关能力由：

- `parseShellFrontmatter()`
- `executeShellCommandsInPrompt()`

组合实现。

这代表 skill 不只是静态说明书，它可以在 prompt 展开前动态取值。

但这里有一个重要限制：

- MCP skill 被明确禁止执行 inline shell block

原因代码里也写明了：

- MCP skill 是远端、不可信来源
- 不能把它当成本地可信脚本来执行

这是非常明确的一道安全边界。

## 7. `paths` 字段让 skill 变成条件定义

`paths` 不影响 skill 内容本身，而影响 skill 是否进入可见集合。

解析时：

- `splitPathInFrontmatter()` 支持字符串或 YAML 数组
- 支持 brace expansion
- `/**` 会被归一化

激活时：

- 用 `ignore` 库按 gitignore 风格匹配

这让一个 skill 可以写成：

- “只在接触 `src/payments/**` 时才暴露”
- “只在编辑 `*.test.ts` 或 `*.spec.ts` 时才暴露”

## 8. plugin skill 的创建方式

plugin skill 的核心逻辑在：

- `restored-src/src/utils/plugins/loadPluginCommands.ts`
- `restored-src/src/utils/plugins/pluginLoader.ts`

### 8.1 plugin 自动技能目录

如果 plugin 根目录下存在：

```text
skills/
```

那么 `pluginLoader.ts` 会把它登记到 `plugin.skillsPath`。

### 8.2 manifest 额外 skill 路径

plugin manifest 还可以显式声明：

```json
{
  "skills": "relative/path"
}
```

或：

```json
{
  "skills": ["a", "b"]
}
```

这些路径会被校验后存进 `plugin.skillsPaths`。

### 8.3 plugin skill 命名

plugin skill 自动加插件名前缀：

- `release-assistant:publish`
- `my-plugin:mobile:release`

这样可以避免和本地 skill 冲名。

## 9. plugin skill 的 prompt 扩展能力比本地 skill 更多

`createPluginCommand()` 在 `getPromptForCommand()` 里额外做了几件事：

1. 替换 `${CLAUDE_PLUGIN_ROOT}`
2. 替换 `${CLAUDE_PLUGIN_DATA}`
3. 从 plugin 的 user config 里替换 `${user_config.xxx}`
4. 对 skill 模式下再替换 `${CLAUDE_SKILL_DIR}`
5. 替换 `${CLAUDE_SESSION_ID}`
6. 执行 shell block

这说明 plugin skill 是比普通本地 skill 更完整的一种“带配置注入的 prompt 组件”。

## 10. bundled skill 的创建方式

bundled skill 不依赖磁盘上的 `SKILL.md`。

它通过：

- `restored-src/src/skills/bundledSkills.ts`
- `registerBundledSkill(definition)`

来注册。

`BundledSkillDefinition` 里直接给出：

- `name`
- `description`
- `allowedTools`
- `context`
- `agent`
- `hooks`
- `getPromptForCommand`

所以 bundled skill 本质上是“代码定义 skill”。

## 11. bundled skill 还能带 reference files

`BundledSkillDefinition.files` 允许 skill 在第一次调用时把内置文件解压到磁盘。

流程是：

1. 首次调用 skill
2. `extractBundledSkillFiles()` 把文件写到一个临时 skill 目录
3. 自动在 prompt 前注入：
   `Base directory for this skill: <dir>`

这样模型就能继续用 Read/Grep 读取 skill 附带的参考文件。

这个设计很聪明，因为它解决了一个问题：

- bundled skill 自己没有真实的源码目录
- 但模型执行 skill 时又需要“旁路读取更多上下文”

## 12. built-in plugin skill 的定位

`restored-src/src/plugins/builtinPlugins.ts` 说明了 built-in plugin skill 和 bundled skill 的差别。

区别不在执行模型，而在管理模型：

- bundled skill 永远内置
- built-in plugin skill 是“内置但可启停”的 plugin 组件

不过转换成 `Command` 时，它仍然会被标成：

- `source: 'bundled'`
- `loadedFrom: 'bundled'`

这么做是为了让它在 SkillTool、analytics、prompt 截断策略上复用 bundled 逻辑。

## 13. bundled skill 的实例

`restored-src/src/skills/bundled/index.ts` 里会注册：

- `verify`
- `skillify`
- `remember`
- `simplify`
- `batch`
- `stuck`

举两个例子：

### `verify`

`restored-src/src/skills/bundled/verify.ts`

它并不是手写 `description + body`，而是：

- 从 `verifyContent.ts` 里的 `SKILL_MD` 解析 frontmatter
- 再复用正文

这说明 bundled skill 也在向 `SKILL.md` 语义靠拢，只是载体从磁盘文件变成了代码常量。

### `skillify`

`restored-src/src/skills/bundled/skillify.ts`

它本身就是一个“生成 skill 的 skill”。

这个 skill 会：

- 读取 session memory
- 读取当前会话里的 user messages
- 指导模型访谈用户
- 最终产出一个新的 `SKILL.md`

这说明项目作者把 skill 当成一类第一等产物，而不是临时功能。

## 14. MCP skill 的创建方式

这里需要明确区分“我确认到的”和“我只能反推的”。

### 我能确认的部分

从这些文件可以确认：

- `services/mcp/client.ts`
- `services/mcp/useManageMCPConnections.ts`
- `services/mcp/utils.ts`
- `skills/mcpSkillBuilders.ts`
- `skills/loadSkillsDir.ts`

可以确定 MCP skill 机制具备这些特征：

1. MCP skill 来自 `skill://` resources，而不是 prompt 列表
2. 它们最终也会变成 `Command`
3. 命名格式是 `<server>:<skill>`
4. `loadedFrom === 'mcp'`
5. 构建过程复用了 `createSkillCommand()` 和 `parseSkillFrontmatterFields()`

### 当前快照里缺失的部分

`restored-src/src/skills/mcpSkills.ts` 不在当前快照里。

所以 MCP skill 如何从 resource body 读取 markdown、如何决定 baseDir、如何缓存资源内容，当前只能根据调用点推断，不能逐行确认。

## 15. 一个实现上的不一致

`frontmatterParser.ts` 的注释说：

- `commands/` 默认 `user-invocable: true`
- `skills/` 默认 `user-invocable: false`

但当前实现并不是这样：

- `parseSkillFrontmatterFields()` 默认 `true`
- `createPluginCommand()` 默认 `true`

所以从“当前 restored-src 行为”来看：

- skill 默认可以被用户直接 `/name` 调用
- 只有显式设 `user-invocable: false` 才会变成模型专用

这很可能是注释滞后，或者设计曾经改过但文档没同步。

## 16. 这一层的本质

skill 的“创建”不是单纯写一个 markdown 文件，而是把某种来源的内容转换成统一的 `PromptCommand`：

- 给它命名
- 给它权限
- 给它资源目录
- 给它参数替换规则
- 给它 shell 扩展规则
- 给它执行上下文

所以 skill creation 的本质不是文件创建，而是“能力注册”。
