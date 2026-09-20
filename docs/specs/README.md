# 规格总览 — AI 辅助研发平台高阶能力

## 全局目标

在 Multica 现有基础上（Agents + Squads + Skills + Issues/Runs + Autopilots + Plugin 系统），
打造一个 **AI 辅助研发平台**，支持：

1. **工作流**（工厂产线 DAG）— 确定性的多步骤编排，支持串/并/分支/循环/人工门禁
2. **小应用**（App 发布）— 将小队、工作流、或小队+工作流打包发布成直接被他人使用的工具
3. **DIY 能力**（万物皆插件）— 从"小队/工作流灵活组合"到"万物皆插件"的递进扩展
4. **可视化运行进展**— DAG 画布实时状态 + 小队讨论线程 + 执行时间线，UI 足够美观

后台对接 Claude CLI（及 25 个其他 agent CLI），复用 Multica 现有的 daemon + runtime + agent_task_queue 机制。

## 现有基础

Multica 已有的 12 个核心原语不需要重写，工作流/小应用/DIY 都应作为**新增层**嫁接上来：

| 原语 | 作用 | 复用方式 |
|---|---|---|
| Agent（26 CLI runtime） | 执行单元 | 工作流的 Agent-Call 节点直接调用 `CreateAgentTask` |
| Squad（leader 路由） | 多 agent + 人工协作 | 工作流的 Squad-Call 节点调用现有 leader 路由（含终止条件 + 结构化决策输出） |
| Skill（SKILL.md playbook） | 可复用 agent 能力 | 工作流节点的配置模板；插件的 resource 类型 |
| Issue（工作项） | 工作目标 + 讨论 + 状态 | 工作流可绑定到 Issue；App 调用可创建 Issue |
| Run（代码：Task，`agent_task_queue`） | 一次 agent 执行 | 工作流执行复用 Run 的执行日志 + token 统计 |
| Autopilot（cron/webhook 触发） | 定时/事件触发 | 工作流触发器复用 Autopilot 的 cron/webhook 机制 |
| Project（issue 分组 + 资源绑定） | 工作分组 + 上下文 | App 可关联到 Project |
| Chat（一对一私聊） | 非 issue 对话 | App 可暴露为 chat 入口 |
| Inbox（通知中心） | 人工通知 | Human-Input 节点暂停时通知到 Inbox |
| Plugin 系统（manifest + hooks + surfaces） | 扩展基础设施 | 工作流节点/触发器/App 模板作为新 `contributes` 类型 |
| Board UI（看板 + 执行日志） | 可视化基底 | 扩展为 DAG 画布 + 运行时进度 |
| Issue.stage 屏障（`parent_issue_id` + `stage`） | 有序子 issue 屏障 | issue-centric 工作流可复用屏障原语 |

## 规格拆分

| # | 规格 | 核心内容 | 借鉴 |
|---|---|---|---|
| 01 | [工作流引擎](./01-workflow-engine.md) | DAG 定义 + 执行引擎 + 节点类型 + 变量传递 + 暂停/恢复 + 静态模板编辑器 | Dify / n8n / NocoBase / LangGraph |
| 02 | [App 管理](./02-app-management.md) | App 实体 + 发布面 + 访问控制 + 运行时状态机 | Dify / n8n / NocoBase |
| 03 | [DIY 插件扩展](./03-diy-plugin-extensibility.md) | Manifest v2 + 新 contributes 类型 + reverse calls + 分发 | Dify / NocoBase / n8n |
| 04 | [可视化运行进展](./04-visual-run-progress.md) | 运行时 DAG 画布 + 节点状态 + 变量检查 + 小队讨论可视化 + 执行时间线 | Dify / n8n / NocoBase |

每个规格独立可开发，但共享全局目标。**实施顺序**：01（工作流引擎）→ 02（App 管理）→ 04（可视化运行进展）→ 03（DIY 插件扩展，最后实现）。

> **关于"小队 + 工作流 + 多 Agent 组合"**：这不是独立规格——组合本身就是包含 Squad-Call 节点的 workflow，发布出来就是一个 App。最复杂的场景（squad 讨论 → workflow 编码 → finalize）= 一个 workflow，里面有三个大节点：Squad-Call（讨论）→ Sub-Workflow（编码 DAG）→ Squad-Call（收尾）。Squad-Call 节点的设计（终止条件、结构化决策输出、卡顿检测）在 [01-工作流引擎](./01-workflow-engine.md) 中定义。

## 关键设计原则

1. **嫁接不重写** — 每个新能力都作为现有插件系统 `contributes` 的新类型，不另起炉灶
2. **Issue-first** — Multica 是 issue-driven，不是 chatbot-first；工作流/App 围绕 Issue 组织
3. **类型化交接** — 模块间传递类型化 artifact（zod schema），非自由文本
4. **可审计** — 一次工作流执行 = 一个可回溯的 case，每步状态可 checkpoint
5. **Code never leaves it** — 插件作为 daemon subprocess 运行，不走 serverless
