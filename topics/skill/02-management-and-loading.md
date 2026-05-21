# 02. 管理与装载机制

## 1. 启动时的 skill 注册入口

启动阶段最重要的入口在 `restored-src/src/main.tsx`。

这里有两个关键动作：

1. `initBuiltinPlugins()`
2. `initBundledSkills()`

并且它们被刻意放在 `getCommands()` 之前调用。代码注释说明得很清楚：

- bundled skills 和 built-in plugins 都是纯内存注册
- 如果晚于 `getCommands()`，并行启动会拿到空列表

这说明 skill 机制不是“懒加载后自然可见”，而是强依赖启动顺序。

## 2. 本地 skill 的搜索范围

本地 file-based skill 主要由 `restored-src/src/skills/loadSkillsDir.ts` 负责，而它依赖 `restored-src/src/utils/markdownConfigLoader.ts` 的目录搜索规则。

`getProjectDirsUpToHome(subdir, cwd)` 的行为是：

- 从当前目录一路向上找 `.claude/<subdir>`
- 如果当前目录在 git 仓库里，就在 git root 停止
- 如果不在 git 仓库里，就最多走到 home 目录
- 不会把 git 仓库外层父目录的 `.claude` 泄漏进来

这意味着项目 skill 的可见范围是：

- 当前目录到 git root 之间所有已有的 `.claude/skills`
- 以及 `--add-dir` 注入的额外目录
- 再加上用户级和受策略管理的 skill 目录

## 3. skill 的多来源装载顺序

`getSkillDirCommands(cwd)` 会并行收集以下来源：

### 3.1 managed skills

目录：

- `getManagedFilePath()/.claude/skills`

来源标记：

- `source: 'policySettings'`

### 3.2 user skills

目录：

- `~/.claude/skills`

来源标记：

- `source: 'userSettings'`

### 3.3 project skills

目录：

- 从当前目录向上到 git root 的所有 `.claude/skills`

来源标记：

- `source: 'projectSettings'`

### 3.4 additional dirs

目录：

- `--add-dir` 对应目录下的 `.claude/skills`

来源标记：

- 仍然归到 `projectSettings`

### 3.5 legacy commands as skills

目录：

- `.claude/commands`

来源标记：

- `loadedFrom: 'commands_DEPRECATED'`

这部分说明 skill 系统兼容两套历史写法：

- 新式：`.claude/skills/<name>/SKILL.md`
- 旧式：`.claude/commands/*.md` 或嵌套目录中的 `SKILL.md`

## 4. `.claude/skills` 的装载规则

`loadSkillsFromSkillsDir()` 非常严格，只接受目录式 skill：

- `skill-name/SKILL.md`

它明确不支持：

- 直接把单个 `.md` 文件丢在 `skills/` 根目录

这和 legacy `commands/` 的容忍度不同。

## 5. `.claude/commands` 的兼容规则

`loadSkillsFromCommandsDir()` 支持两种格式：

1. 单文件命令
   `commands/foo.md`
2. 目录式 skill
   `commands/foo/SKILL.md`

其中 `transformSkillFiles()` 会做一件很关键的事：

- 如果一个目录里存在 `SKILL.md`
- 那么这个目录下其他 markdown 文件都被忽略
- 只把该目录视为一个 skill 容器

这保证 legacy 目录结构能逐步向 skill 目录结构迁移，而不会重复加载。

## 6. 命名规则

### 本地 `.claude/skills`

名字直接取目录名：

- `.claude/skills/review/SKILL.md` -> `review`

### legacy `.claude/commands`

如果是普通 markdown：

- `commands/git/commit.md` -> `git:commit`

如果是目录式 skill：

- `commands/release/SKILL.md` -> `release`
- `commands/mobile/release/SKILL.md` -> `mobile:release`

### plugin skill

plugin loader 会自动加插件前缀：

- `my-plugin:review`
- `my-plugin:release:prepare`

### MCP skill

从 `services/mcp/utils.ts` 可以看出，MCP skill 的名字格式被约定为：

- `<server>:<skill>`

它和 MCP prompt 的：

- `mcp__<server>__<prompt>`

是两套不同命名体系。

## 7. 去重机制

### 7.1 file-based skills 的去重

`loadSkillsDir.ts` 使用 `realpath()` 做文件身份识别。

目的：

- 解决符号链接重复加载
- 解决重叠父目录重复扫描
- 解决同一物理文件通过不同路径被发现

去重策略是“先到先得”。

### 7.2 plugin skills 的去重

`loadPluginCommands.ts` 使用 `loadedPaths` 和 `isDuplicatePath()` 避免：

- 同一个 plugin 内不同配置路径重复装载同一 skill

### 7.3 markdown loader 的去重

`markdownConfigLoader.ts` 在更通用的层面也会去重，避免用户目录和项目目录通过 symlink 指向同一物理文件时重复出现。

## 8. 缓存机制

skill 管理大量使用 `memoize()`：

- `getSkillDirCommands`
- `loadAllCommands`
- `getSkillToolCommands`
- `getSlashCommandToolSkills`
- plugin skill/command loaders

缓存失效函数分为两类：

### 8.1 只清命令聚合缓存

`clearCommandMemoizationCaches()`

用途：

- 当动态 skill 被发现后，只刷新命令视图
- 不抹掉已经发现的动态 skill 状态

### 8.2 全量清理

`clearCommandsCache()`

会继续联动：

- `clearPluginCommandCache()`
- `clearPluginSkillsCache()`
- `clearSkillCaches()`

## 9. 动态发现机制

这部分是 skill 机制里非常“产品化”的一段。

相关入口：

- `discoverSkillDirsForPaths()`
- `addSkillDirectories()`
- `activateConditionalSkillsForPaths()`

触发点来自文件工具：

- `FileReadTool`
- `FileWriteTool`
- `FileEditTool`

### 9.1 动态发现做了什么

当模型读取、写入、编辑某个文件时，系统会：

1. 从该文件所在目录向上走到 cwd
2. 查找中途是否存在新的 `.claude/skills`
3. 如果存在，把这些目录加入动态 skill 发现结果
4. 异步装载并并入 `dynamicSkills`

### 9.2 为什么只扫 cwd 之下

因为 cwd 层级本身的 `.claude/skills` 已经在启动时加载了。

动态发现只解决一个问题：

- 当模型第一次“深入”某个子目录时，才把那个子目录局部 skill 引入进来

这让 skill 可以和具体代码区域绑定，而不是一启动就把所有嵌套 skill 全灌进来。

### 9.3 gitignored 目录不会被动态 skill 悄悄注入

`discoverSkillDirsForPaths()` 在发现 `.claude/skills` 后还会调用 `isPathGitignored()`。

目的：

- 阻止例如 `node_modules/pkg/.claude/skills` 这种路径被偷偷装入

这是个很关键的信任边界。

## 10. conditional skills

`paths` frontmatter 让 skill 变成“条件激活”。

处理流程是：

1. 启动时先把带 `paths` 的 skill 放入 `conditionalSkills`
2. 不立即加入可用 skill 列表
3. 当文件工具操作某个路径时
4. `activateConditionalSkillsForPaths()` 用 `ignore` 库按 gitignore 风格匹配
5. 匹配成功才把该 skill 移到 `dynamicSkills`

因此 `paths` 不是运行时过滤，而是“延迟曝光”机制。

这意味着：

- skill 在命中文件之前，模型根本看不到它
- 一旦命中，就进入动态 skill 集合

## 11. hot reload 机制

`restored-src/src/utils/skills/skillChangeDetector.ts` 负责监听 skill/command 文件变化。

它会监控：

- `~/.claude/skills`
- `~/.claude/commands`
- 当前项目 `.claude/skills`
- 当前项目 `.claude/commands`
- `--add-dir` 对应目录的 `.claude/skills`

核心特征：

- 使用 `chokidar`
- 做了 write-finish 稳定等待
- 用 debounce 合并短时间大量文件变更
- 先执行一次 `ConfigChange` hook
- 如果 hook 阻止，则不 reload
- 否则清理 skill/command 缓存并发出 `skillsChanged` 信号

这里能看到 skill 管理已经和 hook 体系、设置变更体系融合了。

## 12. `/skills` 菜单到底展示什么

`/skills` 命令只是个 local-jsx 包装，真正 UI 在：

- `restored-src/src/components/skills/SkillsMenu.tsx`

它只展示：

- `loadedFrom === 'skills'`
- `loadedFrom === 'commands_DEPRECATED'`
- `loadedFrom === 'plugin'`
- `loadedFrom === 'mcp'`

也就是说：

- 展示本地 file-based skills
- 展示 legacy commands-based skills
- 展示 plugin skills
- 展示 MCP skills
- 不展示 bundled skills
- 不展示 built-in plugin skills

这和“SkillTool 能看到什么”不是同一个集合。

## 13. skill 在系统里还有两个“被动曝光”通道

### 13.1 `skill_listing` attachment

`utils/attachments.ts` 会把新增的可用 skill 以 attachment 的形式发给模型。

特点：

- 会按 agent 维度记录哪些名字已经发过
- 初次发送和动态新增会分开
- 用 `formatCommandsWithinBudget()` 控制字数预算
- 开启 skill search 时，会把 turn-0 主动列表收缩成 bundled + MCP，降低噪音

### 13.2 skill discovery

`constants/prompts.ts` 和 attachment 系统一起告诉模型：

- 当前 turn 已经有哪些相关 skill
- 如果当前列表不够，再调用 DiscoverSkills 工具

所以 skill 管理不只是“能不能调用”，还包括“什么时候把 skill 暴露给模型”。

## 14. telemetry 埋点很多

围绕 skill 管理的 telemetry 很多，说明这套系统是被强监控的：

- `tengu_skill_loaded`
- `tengu_dynamic_skills_changed`
- `tengu_skill_file_changed`
- `tengu_dir_search`
- skill description truncation 相关事件

这些埋点说明产品很关心：

- skill 是否被加载
- skill 列表是否太长
- 动态 skill 是否真的有用
- 用户目录扫描成本

## 15. 这一层的本质

如果只看表面，skill 好像只是“读一个 markdown 文件”。

但管理层实际做的是：

- 目录治理
- 来源治理
- 动态治理
- 权限治理
- 热更新治理
- 曝光治理

所以 skill 在这个项目里更接近“一个受控的 prompt capability registry”，而不是简单的 markdown 模板仓库。
