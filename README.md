# learn-claude-code

基于 `@anthropic-ai/claude-code@2.1.88` sourcemap 还原代码所做的源码分析笔记。**不是官方文档**，是个人逆向阅读的产物，行号对应当前快照版本。

## 目录结构

文档分两层——先看概览建立框架，再按需深入专题：

```
.
├── overview/   架构概览：13 篇广度优先的子系统概览
└── topics/     深度专题：7 个多篇深度拆分的专题目录
```

- **`overview/`** —— 全局视角，每篇聚焦一个子系统，快速建立心智模型。
- **`topics/`** —— 对某个机制做多篇逐行精读。建议先读 `overview` 建立框架，再挑感兴趣的 topic 深入。

## `overview/` —— 架构概览（13 篇）

claude-code 全局视角，每篇聚焦一个子系统：

- 00 总览与目录
- 01 核心架构与启动流程
- 02 工具系统架构
- 03 服务层与 API 通信
- 04 权限系统与安全机制
- 05 命令系统与 UI 组件
- 06 Ink 终端渲染引擎
- 07 API 通信与 Query 引擎
- 08 任务管理与 Agent 编排
- 09 记忆系统与技能系统
- 10 配置系统与 Git 集成
- 11 SDK 编程接口与会话管理
- 12 认证系统与插件

## `topics/` —— 深度专题（7 个）

| 专题 | 文档数 | 内容 |
|------|--------|------|
| [`topics/todolist/`](./topics/todolist/) | 8 | TodoList V1/V2、原生工具定义、工具传递与 `tool_use` 数据流 |
| [`topics/agent/`](./topics/agent/) | 11 | 子代理 5 种 spawn 模式 / 并行调度 / 模型选择 / runAgent / fork + worktree 隔离 / agent memory / 生命周期 |
| [`topics/prompt-injection/`](./topics/prompt-injection/) | 8 | 威胁模型 / `<system-reminder>` 标签 / FileRead 双护栏 / Unicode 清洗 / 友方注入预算 / 外部工具隔离 |
| [`topics/messages-pipeline/`](./topics/messages-pipeline/) | 10 | 消息类型 / 流式事件 / normalize 主循环 / 合并 smoosh hoist / tool_reference 协议事故 / 协议合规过滤 / attachment normalizer |
| [`topics/context-management/`](./topics/context-management/) | 8 | 上下文管理与压缩机制深度分析 |
| [`topics/core-models/`](./topics/core-models/) | 9 | 7 个核心实体 model 逐字段精读 + 设计哲学 |
| [`topics/skill/`](./topics/skill/) | 16 | skill 模块专题：结论速查 4 篇 + guided-tour 渐进式教程 11 篇 |

### `topics/todolist/`

TodoList 完整实现链路：工具定义如何暴露给模型、模型如何返回 `tool_use`、runtime 如何执行并持久化，以及 V1 `TodoWrite` 与 V2 `Task*` 两套实现的差异。

### `topics/agent/`

AgentTool 子代理专题（README + 00~09 共 11 篇）：5 种派发模式、并行/串行调度、模型选择五级优先级、`runAgent` 共享/克隆/隔离矩阵、fork + worktree 隔离、agent memory 三层 scope、生命周期与端到端模拟。

### `topics/prompt-injection/`

prompt-injection 防御专题（README + 00~06 共 8 篇）：6 层纵深防御、`<system-reminder>` 标签机制、FileRead 行号前缀 + malware reminder 双护栏、Unicode 清洗管线、友方注入预算、外部工具内容隔离、元防御指令与端到端模拟。

### `topics/messages-pipeline/`

消息规范化管线专题（README + 00~08 共 10 篇）：消息类型与构造、流式事件与 assistant 增量构造、`normalizeMessagesForAPI` 主循环、合并 smoosh hoist、tool_reference 协议边界事故（#21049）、协议合规过滤层、attachment normalizer、端到端模拟与 feature gate 时间线。

### `topics/context-management/`

上下文管理这条主线的深度展开：

- 00 总览与目录
- 01 系统提示构建与消息上下文
- 02 上下文压缩与自动压缩机制
- 03 提示缓存与内容替换机制
- 04 消息处理流水线与请求编排
- 05 完整系统提示词示例
- 06 脏设计与特殊兼容机制总结
- 07 完整数据流示例（三轮对话）

### `topics/core-models/`

7 个核心实体 model 的逐字段精读 + 设计哲学综合，目标是"通过模型设计反推 claude-code 的思考方式"：

- 00 模型地图（7 个 model 的关系总图）
- 01 Command — 命令的统一抽象（PromptCommand / LocalCommand / LocalJSXCommand）
- 02 Tool — 工具的统一抽象（含 ToolUseContext 50+ 字段信封）
- 03 Message — 消息系统（6 大类 + 14 SystemMessage subtype + 30+ Attachment）
- 04 AppState — 全局状态模型（95+ 字段 / DeepImmutable 边界）
- 05 Permission — 权限规则与决策（rules / decisions / 11 reasons）
- 06 Hook — 生命周期钩子（27 events + 13 特化 schema）
- 07 Plugin — 插件清单与加载结果（30+ PluginError type-first）
- 08 设计哲学（12 条可复用设计原则总结）

### `topics/skill/`

skill 模块专题，分两层：

**结论型速查（4 篇）** — 已经熟悉源码、想快速查某个细节时用：

- 01 架构总览
- 02 管理与装载机制
- 03 创建方式与定义格式
- 04 使用与执行机制

**渐进式教程 `guided-tour/`（11 篇）** — 第一次系统读这块代码时用，每一步配精确文件 + 行号：

- 00 学习路线图
- 01 从一个例子开始（verify skill 端到端追踪）
- 02 类型系统先行（Command 联合类型逐字段）
- 03 装载层逐行精读（loadSkillsDir.ts 1086 行分 6 段）
- 04 命令聚合总线（commands.ts 4 路 → 3 视图）
- 05 用户调用链路追踪（/skill 输入到 prompt 注入）
- 06 模型调用链路追踪（SkillTool.ts 1108 行）
- 07 动态发现与条件激活
- 08 热更新与缓存失效
- 09 编一个自己的 skill（动手验证）
- 10 常见疑惑与陷阱

## 一些说明

- 分析基于 `claude-code@2.1.88` 的还原源码（`restored-src/src/...`），版本升级后行号会漂移。
- 这是个人阅读笔记，包含主观判断与"为什么这么设计"的推断，不全是 1:1 的代码描述。遇到拿不准的地方文档里会标注"反推"。
- 如果你看到事实错误或行号错位，欢迎提 issue。
