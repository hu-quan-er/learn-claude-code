# restored-src skill 机制深度分析

本文档分析的是 `claude-code-sourcemap/restored-src` 里的 skill 实现，不是官方文档，也不是对 Claude Code 概念层的泛泛介绍。

重点结论先说：

1. 这个项目里 skill 不是独立的一套对象模型，本质上是 `type: 'prompt'` 的 `Command`。
2. skill 的核心不是“菜单项”，而是“可展开成一段 prompt 的命令”，再叠加 `allowedTools`、`context`、`hooks`、`paths` 等元数据。
3. skill 来源不止一种，至少有本地 `.claude/skills`、遗留 `.claude/commands`、plugin skills、bundled skills、built-in plugin skills、MCP skills、动态发现 skills、conditional skills。
4. “管理”这一层做得很重，包含启动时注册、目录搜索、去重、缓存、热更新、动态发现、按路径激活、权限策略和 telemetry。
5. “使用”这一层分成两条主链路：用户输入 `/xxx` 走 `processSlashCommand.tsx`，模型主动调用走 `SkillTool.ts`。
6. skill 执行结果不会只是临时 prompt，它还会影响本轮工具权限、模型、effort、hook 注册，以及 compact 之后的技能保留。

## 文档结构

- `01-architecture-overview.md`
  统一抽象、核心字段、source/loadedFrom 语义、命令装载顺序。
- `02-management-and-loading.md`
  skill 的管理、目录扫描、缓存、热更新、动态发现、conditional 激活、`/skills` 菜单。
- `03-creation-and-definition.md`
  skill 的创建方式、`SKILL.md` 约定、frontmatter 字段、plugin/bundled/built-in plugin/MCP 来源。
- `04-usage-and-execution.md`
  skill 的执行机制、权限、fork 模式、hook、compact 保留、attachment 提示。

## 建议阅读顺序

1. 先读 `01-architecture-overview.md`
2. 再读 `02-management-and-loading.md`
3. 然后读 `03-creation-and-definition.md`
4. 最后读 `04-usage-and-execution.md`

## 关键源码入口

- `restored-src/src/types/command.ts`
- `restored-src/src/commands.ts`
- `restored-src/src/skills/loadSkillsDir.ts`
- `restored-src/src/skills/bundledSkills.ts`
- `restored-src/src/utils/plugins/loadPluginCommands.ts`
- `restored-src/src/utils/processUserInput/processSlashCommand.tsx`
- `restored-src/src/tools/SkillTool/SkillTool.ts`
- `restored-src/src/utils/skills/skillChangeDetector.ts`

## 我在这份代码里确认到的两个实现细节

### 1. `/skills` 菜单不会展示 bundled skills

`restored-src/src/components/skills/SkillsMenu.tsx` 只过滤：

- `loadedFrom === 'skills'`
- `loadedFrom === 'commands_DEPRECATED'`
- `loadedFrom === 'plugin'`
- `loadedFrom === 'mcp'`

这意味着 bundled skills 和 built-in plugin skills 虽然会进入 `getCommands()`，也会进入 `SkillTool` 的可用列表，但不会出现在 `/skills` 菜单里。

### 2. `user-invocable` 的注释和实现存在不一致

`restored-src/src/utils/frontmatterParser.ts` 的注释写的是：

- `commands/` 默认 `true`
- `skills/` 默认 `false`

但实际实现里：

- `parseSkillFrontmatterFields()` 默认值是 `true`
- `createPluginCommand()` 默认值也是 `true`

按当前 `restored-src` 快照，file-based skill 和 plugin skill 都默认可被用户 `/skill-name` 直接调用，除非 frontmatter 显式设为 `false`。

## 一个需要说明的快照缺口

当前 `restored-src` 快照里看不到 `restored-src/src/skills/mcpSkills.ts`，但有多处代码引用它：

- `restored-src/src/services/mcp/client.ts`
- `restored-src/src/services/mcp/useManageMCPConnections.ts`

所以文档里关于 MCP skill 的“最后一段资源转命令”的细节，是基于调用方、builder 注册点和周边约束反推出来的，不是直接逐行读取该文件得出的。
