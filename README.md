# learn-claude-code

基于 `@anthropic-ai/claude-code@2.1.88` sourcemap 还原代码所做的源码分析笔记。**不是官方文档**，是个人逆向阅读的产物，行号对应当前快照版本。

## 目录

```
.
├── analysis-docs/                  整体架构与各子系统概览（16 篇，含 TodoList / AgentTool / prompt-injection / messages-pipeline 拆分专题）
├── context-management-analysis/    上下文管理与压缩机制深度分析（8 篇）
├── skill-analysis-docs/            skill 模块专题
│   ├── 01~04                       结论型速查（4 篇）
│   └── guided-tour/                带读源码的渐进式教程（11 篇）
└── core-models-analysis/           核心实体模型与设计思想（9 篇）
```

### `analysis-docs/`

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
- 13 TodoList 实现机制（入口页，细节拆分在 `analysis-docs/todolist/`）
- 14 AgentTool 子代理专题（拆分在 `analysis-docs/agent/`，10 篇）
- 15 prompt-injection 防御专题（拆分在 `analysis-docs/prompt-injection/`，7 篇）
- 16 messages-pipeline 消息规范化管线专题（拆分在 `analysis-docs/messages-pipeline/`，9 篇）

### `context-management-analysis/`

上下文管理这条主线的深度展开：

- 00 总览与目录
- 01 系统提示构建与消息上下文
- 02 上下文压缩与自动压缩机制
- 03 提示缓存与内容替换机制
- 04 消息处理流水线与请求编排
- 05 完整系统提示词示例
- 06 脏设计与特殊兼容机制总结
- 07 完整数据流示例（三轮对话）

### `skill-analysis-docs/`

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

### `core-models-analysis/`

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

## 一些说明

- 分析基于 `claude-code@2.1.88` 的还原源码（`restored-src/src/...`），版本升级后行号会漂移。
- 这是个人阅读笔记，包含主观判断与"为什么这么设计"的推断，不全是 1:1 的代码描述。遇到拿不准的地方文档里会标注"反推"。
- 如果你看到事实错误或行号错位，欢迎提 issue。
