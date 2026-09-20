# 03 — DIY 插件扩展

> **全局目标**：AI 辅助研发平台，支持 DIY 自定义 App 发布可复用的能力，后台对接 Claude CLI（现有 Multica 机制）。
> 本规格定义"万物皆插件"的扩展路径：从现有插件系统扩展到覆盖工作流节点、触发器、App 模板、新 UI surface。

## 现有基础：Multica 插件系统已经有什么

Multica 已有**成熟的插件系统**，架构正确，差距在 `contributes` 类型的广度。

### 三个贡献方向（已有）

| 方向 | 含义 | 现有能力 |
|---|---|---|
| **Action**（plugin → host） | 插件调用 host 能力 | `multica.context.get()` / `multica.issue.get()` / `multica.issue.comment()` / `multica.storage.user|workspace` / `multica.ui.resize()` |
| **Hook**（host → plugin） | host 回调插件 | 4 种触发器（ui / manual / event / schedule）+ HTTP/MCP 传输 + 签名验证 + 回调令牌 + 幂等 |
| **Resource**（静态贡献） | 静态内容 | `skill` 类型（插件贡献 SKILL.md，`UpsertPluginSkill`，已有 `skill.plugin_installation_id`） |

### 基础设施（全部已有）

- `multica.plugin.json` manifest（manifest_version / key / name / scopes / config / contributes）
- `plugin_package` + `plugin_package_version`（不可变版本化发布）+ `plugin_package_file`
- `plugin_installation`（consent-based 授权 + granted_scopes + config + token）
- `plugin_hook_schedule`（durable cron + generation epoch）
- `plugin_invocation`（遥测 + 断路器）
- `plugin_storage`（KV + 配额，user/workspace scope）
- `plugin_secret`（加密 + write-only）
- CSP `connect-src` 从 scopes 自动生成
- SSRF 防护（private address 拒绝）
- 5 个示例插件（hello-panel / release-checklist / schedule-pulse / triage-notify / deploy-sentinel）

### Config 字段类型（已有）

`string` / `bool` / `multiline` —— 需要扩展

## 要做成什么样子：扩展路径

### 扩展一：新 Surface 类型（UI 扩展）

今天只有 `issue_panel`（Issue 侧栏 iframe 面板）。要加的：

| 新 Surface 类型 | 用途 | 借鉴 |
|---|---|---|
| `board_view` | 自定义看板视图（自定义列、自定义卡片渲染） | n8n CanvasNodeDefault；NocoBase BlockModel |
| `sidebar_nav` | 侧栏导航项（点击进入插件自定义页面） | NocoBase 菜单页 |
| `settings_tab` | 设置页 tab（workspace 级配置） | Dify 插件设置页 |
| `agent_panel` | Agent 详情面板（beyond issue_panel，显示 agent 专属信息） | Dify NodePanel |
| `run_progress_widget` | Run 进度 widget（自定义运行进展展示） | Dify RunPanel |
| `workflow_editor_panel` | 工作流编辑器的自定义节点配置面板 | n8n INodeProperties displayOptions |
| `app_input_form` | App 调用页的自定义输入表单 | NocoBase uiSchema |

### 扩展二：新 Resource 类型（静态贡献）

今天只有 `skill`。要加的：

| 新 Resource 类型 | 用途 | 借鉴 |
|---|---|---|
| `workflow_node` | 插件贡献自定义工作流节点类型（如"Slack 通知"/"Jira 创建"/"数据库查询"）。声明 input_schema / output_schema / transport / config_schema | Dify ToolNode + ToolParameter；n8n community node；NocoBase registerInstruction |
| `workflow_trigger` | 插件贡献自定义工作流触发器（如"GitHub PR 创建"/"Slack 命令"/"数据库变更"）。声明 events / transport / schedule | Dify TriggerProvider；n8n webhook/poll trigger；NocoBase CollectionTrigger/ScheduleTrigger |
| `app_template` | 插件贡献完整 App 模板（workflow 定义 + squad 配置 + issue 模板 + 访问策略，打包可一键安装） | Dify marketplace plugin；n8n .n8np package；NocoBase plugin |
| `agent_preset` | 插件贡献预配置的 Agent（runtime config + skills + instructions，一键创建） | Dify Agent Strategy plugin |
| `squad_template` | 插件贡献预配置的 Squad（leader + members + instructions，一键创建） | CrewAI Crew template |
| `issue_status_set` | 插件贡献自定义 Issue 状态集合（自定义状态 + 生命周期类别映射 + 转换规则） | NocoBase workflow status |

### 扩展三：新 Config 字段类型

今天只有 `string` / `bool` / `multiline`。要加的：

| 新类型 | 用途 | 借鉴 |
|---|---|---|
| `select` / `multiselect` | 下拉单选 / 多选 | Dify ToolParameter.type SELECT |
| `agent_selector` | 选择 workspace 中的 Agent | Dify ToolParameter.type APP_SELECTOR |
| `squad_selector` | 选择 workspace 中的 Squad | — |
| `workflow_selector` | 选择已发布的工作流 | — |
| `json_schema` | JSON Schema 编辑器（定义 input/output schema） | Dify output_schema |
| `code_editor` | 代码编辑器（JavaScript / Python） | n8n Code node |
| `key_value_list` | 键值对列表（可增删行） | n8n fixedCollection |
| `conditional` | 条件显示（`displayOptions.show`，当其他字段满足条件时显示） | n8n INodeProperties displayOptions |

### 扩展四：新 Hook 事件类型

今天有 `issue.created` / `issue.updated` / `issue.status_changed` / `comment.created` / `task.started` / `task.completed` / `task.failed`。要加的：

| 新事件 | 触发时机 |
|---|---|
| `workflow.started` | 工作流执行开始 |
| `workflow.node_started` | 工作流节点开始执行 |
| `workflow.node_finished` | 工作流节点完成 |
| `workflow.paused` | 工作流暂停在 Human-Input 节点 |
| `workflow.resumed` | 工作流恢复执行 |
| `workflow.finished` | 工作流执行完成 |
| `app.published` | App 发布 |
| `app.invoked` | App 被调用 |
| `squad.leader_evaluated` | Squad leader 完成一次评估 |

### 扩展五：Reverse Call（plugin → host 回调）

今天 Action API 只能读 issue + 写评论。要扩展的：

| Reverse Call | 用途 | 借鉴 |
|---|---|---|
| `multica.issue.create()` | 插件创建 Issue（分配给 agent/squad） | Dify reverse call：plugin 创建 workflow run |
| `multica.agent.trigger()` | 插件触发 Agent 执行 | Dify reverse call：plugin 调用 model/tool/app |
| `multica.workflow.invoke()` | 插件调用已发布的工作流 | Dify reverse call |
| `multica.squad.mention()` | 插件 @-mention 一个 squad | — |
| `multica.workspace.query()` | 插件查询 workspace 数据（issues/agents/squads/skills） | — |
| `multica.task.cancel()` | 插件取消一个正在执行的任务 | — |

### 扩展六：分发路径

今天是 workspace-private（plugin_package 只在本 workspace 安装）。要加的：

| 分发方式 | 机制 | 借鉴 |
|---|---|---|
| **GitHub URL 安装** | `owner/repo` → 拉取 `multica.plugin.json` + 打包文件 → 安装 | Dify `Github` 安装源 |
| **公共 marketplace** | 未来的公共插件目录（需 review/reporting/takedown） | Dify marketplace；n8n templates |
| **插件签名** | 仓库 deploy key 签名 = "verified"；未签名 = warning + opt-in | Dify 签名验证 |

## 借鉴开源组件的哪些能力

### 从 Dify 借鉴

| 能力 | 借鉴点 |
|---|---|
| `manifest.yaml` PluginDeclaration schema | 插件 manifest 的结构化声明（category / resource.permission / meta.runner） |
| `ToolParameter.form`（schema / form / llm） | **最值得借鉴**：三层作者权分离 —— skill 作者固定（schema）/ 工作流设计者配置（form）/ agent 运行时决定（llm） |
| `output_schema`（JSON Schema） | 工作流节点声明输出 schema，下游节点可引用类型化字段 |
| 插件类型（Tool / Model / Endpoint / AgentStrategy / Datasource / Trigger） | Multica 的 `workflow_node` / `workflow_trigger` resource 类型设计参考 |
| Reverse calls（plugin → host 调用 model / tool / app） | Reverse call 扩展的设计参考 |
| 安装源（Github / Marketplace / Package / Remote） | 分发路径设计 |
| 签名验证（certified / unsafe warning） | 插件信任模型 |

### 从 NocoBase 借鉴

| 能力 | 借鉴点 |
|---|---|
| Plugin 生命周期（afterAdd / beforeLoad / load / install / upgrade / beforeEnable / afterEnable / beforeDisable / afterDisable / beforeRemove / afterRemove） | 插件 manifest 的生命周期 hook 设计 |
| `@hapi/topo` 依赖排序（peerDependencies → topo sort） | 插件安装/启用顺序自动解析 |
| `applicationPlugins` DB collection（name / packageName / version / enabled / installed / builtIn / options） | 插件注册表的 `builtIn` 标记 + `options` JSONB |
| `registerInstruction` / `registerTrigger`（插件向 workflow plugin 注册节点类型） | 插件贡献 `workflow_node` resource 的注册机制 |
| 数据模型驱动 UI（Collection → Field → uiSchema → Block） | App 输入表单从工作流 input schema 自动生成 |

### 从 n8n 借鉴

| 能力 | 借鉴点 |
|---|---|
| Community node 分发（npm `n8n-nodes-<name>`，安装到 `~/.n8n/nodes`） | 插件分发命名规范 |
| `INodeProperties`（displayName / name / type / typeOptions / default / displayOptions） | 插件 config 字段的类型化设计 |
| `INodeTypeDescription`（inputs / outputs / properties / credentials / webhooks） | `workflow_node` resource 的声明结构参考 |
| 声明式 vs 程序式节点风格 | 插件节点可以是声明式（requestDefaults + routing）或程序式（execute method） |

## 与 Multica 现有原语的关系

| Multica 原语 | 插件扩展中的角色 |
|---|---|
| `multica.plugin.json` manifest | 扩展 `contributes` 类型 + `config` 字段类型 |
| `plugin_package` + `plugin_package_version` | 分发路径的基础（GitHub URL 安装复用打包机制） |
| `plugin_installation`（consent + scopes） | 新 resource 类型安装时需要 consent（声明新 scopes） |
| `plugin_hook_schedule`（durable cron） | 新事件类型的 hook 调度复用 |
| `plugin_storage` + `plugin_secret` | 插件节点执行时的状态存储 + 密钥管理 |
| `skill.plugin_installation_id` | 已有的 plugin→skill 桥接，是 `workflow_node` resource 的模板 |
| `plugin_invocation`（断路器） | 插件节点执行时的熔断保护 |

## 范围边界

**本规格包含**：
- Manifest v2 扩展（新 `contributes` 类型 + 新 config 字段类型 + 新 hook 事件）
- Reverse call 扩展（plugin 创建 Issue / 触发 Agent / 调用 Workflow）
- 分发路径扩展（GitHub URL 安装 → 未来 marketplace）
- `workflow_node` resource（插件贡献自定义工作流节点）
- `workflow_trigger` resource（插件贡献自定义触发器）
- `app_template` resource（插件贡献完整 App 模板）
- 新 Surface 类型（board_view / sidebar_nav / settings_tab / agent_panel / run_progress_widget）
- 插件签名验证

**本规格不包含**（在其他规格中）：
- 工作流引擎本身 → [01-工作流引擎](./01-workflow-engine.md)
- App 管理 → [02-App 管理](./02-app-management.md)
- 可视化运行进展 → [04-可视化运行进展](./04-visual-run-progress.md)

## 验收标准概要

1. 插件可以贡献自定义工作流节点类型（manifest 声明 `workflow_node` resource，安装后在工作流编辑器节点面板可见）
2. 插件可以贡献自定义工作流触发器（manifest 声明 `workflow_trigger` resource，安装后在工作流触发器配置中可选）
3. 插件可以贡献完整 App 模板（manifest 声明 `app_template` resource，安装后一键创建 App）
4. 新 config 字段类型（select / agent_selector / squad_selector / workflow_selector / json_schema / code_editor / conditional）在插件配置面板正确渲染
5. 新 hook 事件（workflow.* / app.* / squad.*）正确触发已注册的插件 hook
6. 插件可通过 reverse call 创建 Issue / 触发 Agent / 调用已发布工作流
7. 插件可通过 GitHub URL 安装（`owner/repo` → 拉取 manifest + 文件 → consent → 安装）
8. 已签名（deploy key）的插件显示 "verified"；未签名的显示 warning + opt-in
9. 新 Surface 类型（board_view / sidebar_nav / settings_tab / agent_panel / run_progress_widget）在对应位置正确渲染
10. 插件依赖（peerDependencies）在安装/启用时自动排序
