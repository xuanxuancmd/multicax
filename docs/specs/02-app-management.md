# 02 — App 管理

> **全局目标**：AI 辅助研发平台，支持 DIY 自定义 App 发布可复用的能力，后台对接 Claude CLI（现有 Multica 机制）。
> 本规格定义"小应用"：将工作流发布成直接被他人使用的工具。

## 要做成什么样子

一个 **App** 是一个**已发布的工作流**，加上**访问策略**和**暴露面**，使其能被他人直接调用。

> squad 不是 App 的独立字段——squad 通过 workflow 的 **Squad-Call 节点**集成。所以"小队+工作流"组合本质就是一个包含 Squad-Call 节点的 workflow，发布出来就是一个 App。Squad-Call 节点的设计（终止条件、结构化决策输出、卡顿检测）在 [01-工作流引擎](./01-workflow-engine.md) 中定义。

### App 的组成

```
App = {
  workflow: PublishedWorkflow,          // 已发布的工作流（必须）
  // squad 不作为独立字段——squad 通过 workflow 的 Squad-Call 节点集成
  issue_template: IssueTemplate,        // 调用 App 时创建的 Issue 模板
  access_policy: {
    enable_api: bool,                   // 是否暴露 REST API
    enable_share_link: bool,            // 是否暴露分享链接
    enable_embed: bool,                 // 是否允许 iframe 嵌入
    access_mode: 'anyone' | 'authenticated' | 'workspace_members' | 'specific_members',
  },
  branding: { title, description, icon, theme? },
  status: 'draft' | 'published' | 'archived',
}
```

### App 的组合模式

App 本质上是发布一个 workflow。workflow 可以包含不同类型的节点，形成不同类型的 App：

| App 类型 | workflow 内容 | 场景 |
|---|---|---|
| 纯自动化工具 | Agent-Call / Code / HTTP 节点 | 自动化任务（如"代码格式化"、"文档生成"） |
| 小队讨论型 | Squad-Call 节点 | 多 agent + 人工讨论（如"需求评审"、"方案讨论"） |
| 需求开发型 | Squad-Call → Sub-Workflow → Squad-Call | 最复杂场景：squad 讨论澄清需求 → workflow 编码 → squad 收尾 |

> 最复杂的场景（需求开发）= 一个 workflow，里面有三个大节点：
> 1. **Squad-Call 节点**（讨论，产出结构化 `SquadDecision`）
> 2. **Sub-Workflow 节点**（编码 DAG：Plan→Research→Code→Test→Review，消费 `SquadDecision` 作为输入）
> 3. **Squad-Call 节点**（收尾：review/QA/docs agent 轮流执行，聚合成 `FinalizationReport`）
>
> 这不需要独立的"混合编排"概念——它是 workflow 的 Squad-Call + Sub-Workflow 节点的自然组合。

### App 的暴露面

| 暴露面 | 机制 | 调用方式 |
|---|---|---|
| **API** | App-scoped API key | `POST /api/apps/{slug}/invoke` + JSON body → 返回 `task_id`（立即）+ `run_id`（轮询结果） |
| **分享链接** | workspace-scoped URL `/w/{slug}/apps/{app-slug}` | 浏览器打开 → 渲染输入表单 → 提交 → 展示运行进展 |
| **iframe 嵌入** | embed.js + CSP frame-ancestor | 第三方页面嵌入 iframe → postMessage 通信 |
| **Chat 入口** | 复用现有 Chat | App 暴露为 chat 对话入口（Chatflow 模式） |

### App 运行时状态流转图

App 的**调用生命周期**（不是工作流内部节点的状态，而是 App 被外部调用的状态）：

```
                    ┌──────────────────┐
                    │  App 调用状态机   │
                    └──────────────────┘

  [invoked] ──► [queued] ──► [running] ──┬──► [succeeded] ──► [done]
                                          │
                                          ├──► [paused] ──► [resumed] ──► [running]
                                          │     (Human-Input          │
                                          │      节点暂停)             │
                                          │                            │
                                          ├──► [failed] ──► [retry] ──► [running]
                                          │
                                          └──► [cancelled] ──► [done]
```

| 状态 | 含义 | 触发 |
|---|---|---|
| `invoked` | App 被调用（API/链接/embed/chat） | 外部请求 |
| `queued` | 工作流执行入队 | 系统创建 workflow_run |
| `running` | 工作流正在执行 | runtime claim |
| `paused` | 工作流暂停在 Human-Input 节点 | 节点暂停 → 创建 review Issue |
| `resumed` | 人工提交后恢复 | submit action |
| `succeeded` | 工作流成功完成 | 所有节点完成 |
| `failed` | 工作流失败 | 节点失败且无法重试 |
| `retry` | 自动重试中 | 瞬时故障 |
| `cancelled` | 被手动取消 | 调用者取消 |
| `done` | 终态 | succeeded/cancelled |

### App 管理界面

- **App 列表页**：所有 App（draft/published/archived），显示状态、调用次数、最近调用时间
- **App 创建向导**：选择已发布的 workflow → 配置 issue 模板 → 配置访问策略 → 配置 branding → 发布
- **App 详情页**：基本信息 + 调用历史 + 调用统计（次数/成功率/平均耗时/token 消耗）+ 访问策略管理 + API key 管理
- **App 调用页**（分享链接打开时）：渲染输入表单（从工作流的 input schema 自动生成）→ 提交 → 展示运行进展
- **App 嵌入配置页**：生成 embed 代码 + CSP 配置

## 借鉴开源组件的哪些能力

### 从 Dify 借鉴

| 能力 | 借鉴点 |
|---|---|
| App 模型（`mode` / `enable_site` / `enable_api`） | App 实体的核心字段：模式（Workflow / Chatflow）、是否暴露 web / API |
| `access_mode`（anyone / authenticated / all_members / specific_members） | 访问控制四级模型 |
| iframe embed（`embed.js` + CSP `frame-ancestor`） | 安全嵌入第三方页面 |
| API key auth | App-scoped API key，每次调用携带 |
| Draft → Published 快照 | App 版本化：draft 编辑 → published 不可变快照 |
| `enable_site` Site 配置（title / icon / theme） | App 的 branding 配置 |
| MCP server 暴露 | Dify 1.0+ 将 App 暴露为 MCP server —— Multica 可借鉴，App 作为 MCP tool 供 agent 调用 |

### 从 n8n 借鉴

| 能力 | 借鉴点 |
|---|---|
| `.n8np` 包格式（gzip tar + manifest.json + workflows/ + credentials/ + variables/） | App 导出/导入包格式：manifest + workflow 定义 + issue 模板 + 权限配置 |
| 子工作流作为可复用单元（Execute Workflow node, Source: Database/File/Parameter/URL） | App 内部的工作流可以调用另一个已发布的工作流 |
| 环境变量 vs 工作流变量 | App 级别的变量（secrets 独立于 workflow DSL） |
| 凭证不随导出泄露 | App 导出时不包含 API key / secrets |

### 从 NocoBase 借鉴

| 能力 | 借鉴点 |
|---|---|
| `plugin-multi-app-manager`（applications collection: name / displayName / cname / status / options） | 多 App 管理模型 |
| Approval 工作流模式（快照提交记录 + To-do Center） | App 调用需要审批时的模型 |
| 数据模型驱动 UI（Collection → Field → uiSchema → Block） | App 输入表单从工作流的 input schema 自动生成 |

## 与 Multica 现有原语的关系

| Multica 原语 | App 中的角色 |
|---|---|
| `Autopilot.issue_title_template` | App 调用时创建 Issue 的模板来源 |
| `workspace_share_link` | 分享链接的模型基础（但需反转为公共调用面，非 workspace 邀请） |
| Auth token | API key 复用现有 auth token 机制 |
| `Workspace` + roles | 访问控制的 workspace_members / specific_members 基础 |
| `plugin_package` + `plugin_installation` | App 模板可通过插件分发（`app_template` resource 类型） |
| `Issue` + `Run` | App 调用创建 Issue + 工作流执行创建 Run |

## 范围边界

**本规格包含**：
- App 实体定义（已发布 workflow + issue 模板 + 访问策略 + branding；squad 通过 workflow 的 Squad-Call 节点集成）
- App 发布面（API / 分享链接 / iframe embed / chat 入口）
- App 访问控制（四级 access_mode + API key 管理）
- App 运行时状态机（invoked → queued → running → paused → succeeded/failed）
- App 管理界面（列表 / 创建向导 / 详情 / 调用页 / 嵌入配置）
- App 导出/导入包格式

**本规格不包含**（在其他规格中）：
- 工作流引擎本身（含 Squad-Call 节点设计）→ [01-工作流引擎](./01-workflow-engine.md)
- 运行时可视化（调用页的运行进展展示）→ [04-可视化运行进展](./04-visual-run-progress.md)
- 插件贡献 App 模板 → [03-DIY 插件扩展](./03-diy-plugin-extensibility.md)

## 验收标准概要

1. 用户可以从已发布的工作流创建 App（选择 workflow → 配置访问策略 → 发布）
2. App 可通过 API key 调用（`POST /api/apps/{slug}/invoke`），返回 task_id + run_id
3. App 可通过分享链接访问（浏览器打开 → 输入表单 → 提交 → 展示进展）
4. App 可通过 iframe 嵌入第三方页面（embed.js + CSP）
5. App 的访问控制正确执行（anyone / authenticated / workspace_members / specific_members）
6. App 调用创建 Issue + 触发工作流执行，状态流转正确（invoked → queued → running → succeeded/failed/paused）
7. Human-Input 节点暂停时，App 调用者能看到"等待人工审核"状态
8. App 可导出为包（不含 secrets），可在另一个 workspace 导入
9. App 调用历史、统计（次数/成功率/耗时/token）可查看
