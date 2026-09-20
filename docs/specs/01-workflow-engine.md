# 01 — 工作流引擎

> **全局目标**：AI 辅助研发平台，支持 DIY 自定义 App 发布可复用的能力，后台对接 Claude CLI（现有 Multica 机制）。
> 本规格定义"工厂产线"工作流：确定性的多步骤编排引擎。
>
> **决策基础**：23 项架构决策（G1-G19 + 研究综合），详见 `.wayfinder/maps/workflow-engine.md`。
> 8 份研究报告（Dify/n8n/NocoBase/LangGraph/AutoGen/OpenAI SDK 源码分析 + NocoBase vs Dify 扩展性对比）位于 `.wayfinder/research/`。

## Problem Statement

Multica 已有 Agent、Squad、Skill、Issue/Run、Autopilot、Plugin 系统 12 个核心原语，但缺少**确定性的多步骤编排能力**。用户需要将多个 Agent、人工审批、条件分支、子流程组合成可复用的自动化产线，而不是每次手动协调。现有的 Squad leader 路由是非确定性的（leader 自主决策），无法满足需要确定性执行路径的场景（如 CI/CD、代码审查流水线、自动化测试编排）。

## Solution

在 Multica 现有原语上嫁接一个**确定性 DAG 工作流引擎**：JSON DAG 定义 → super-step 执行引擎 → 变量池传递 → 暂停/恢复 → 静态模板编辑器。引擎不重写现有原语，而是作为新的编排层，通过 `CreateAgentTask`、Squad leader 路由、Autopilot 触发器等现有机制执行实际工作。

## 主参考与次参考

> **选型原则**：从 UI 美观度、工作流编排能力、引入复杂度三个维度选择**一个主参考**，使 workflow 快速能力成型。其他组件作为次参考，在特定维度补充。

### 主参考：Dify

| 维度 | 评估 |
|---|---|
| UI 美观度 | ⭐⭐⭐⭐⭐ 最现代、简洁、设计感强 |
| 编排能力 | ⭐⭐⭐⭐ LLM 节点、Agent 节点、变量池、IF/ELSE、Iteration、Human-Input、Code——覆盖 Multica 需求 90% |
| 引入复杂度 | ⭐⭐ 低——React + React Flow = Multica 前端栈完全匹配，JSONB blob + `{{#var#}}` = 简单数据结构 |
| 技术栈匹配 | ✅ React + React Flow + Next.js = Multica 完全一致，前端代码可直接参考 |

### 次参考

| 组件 | 参考什么 | 为什么不选为主参考 |
|---|---|---|
| **LangGraph** | 执行引擎（Pregel super-step + checkpoint + interrupt/resume + Send/reducer） | Python 库，非前端组件；但引擎模型最完整，适配 Go+PostgreSQL |
| **n8n** | execution_entity schema（deduplicationKey + 两阶段删除）+ Wait node resume 机制 | Vue 3 ≠ React，IConnections 嵌套字典 ≠ React Flow edges[] |
| **NocoBase** | Human-Input CAS 并发安全 + 审批模式 + 状态机常量 | 链表+branchIndex 模型不兼容 React Flow；但 DIY/插件扩展性待 R8 研究确认 |
| **AutoGen** | Squad-Call 终止条件 + MagenticOne stall detection | Python 库；但终止条件模型最完整 |
| **OpenAI Agents SDK** | 节点类型化交接契约（input_type/output_type + Guardrails） | Python SDK；但类型化交接模式最干净 |

### 前端参考清单（主参考 Dify）

| Multica 组件 | 参考 Dify 什么 | Dify 源码位置 |
|---|---|---|
| DAG 画布 | React Flow 画布 UX（节点拖拽、边连接、缩放、minimap） | `api/core/workflow/` + 前端 `packages/` |
| 节点面板 | 左侧节点类型拖拽面板 | Dify 前端 |
| 节点配置面板 | 右侧参数表单 + displayOptions 条件显示 | Dify 前端 |
| 变量选择器 | `{{#node_id.var#}}` 下拉选择上游节点输出 | Dify 前端 |
| Draft 自动保存 | debounced PUT 整个 graph JSON | Dify 前端 |
| React Flow 集成 | React Flow + Next.js + packages/views 集成模式 | Dify 前端 |
| 路由 | Multica 现有 `[workspaceSlug]/(dashboard)/` 模式 | Multica `apps/web/app/` |

### 后端参考清单

| Multica 组件 | 主参考 | 参考什么 | 源码位置 |
|---|---|---|---|
| `workflow_definition` 表 | **Dify** `workflows` 表 | graph JSONB blob + version draft/published + environment_variables 独立列 | Dify `api/models/workflow.py` |
| `workflow_run` 表 | **Dify** `WorkflowRun` + Multica `autopilot_run` | status CHECK + result JSONB + graph 快照复制 | Dify + Multica `server/migrations/042_autopilot.up.sql` |
| `workflow_node_execution` 表 | **Multica** `task_message` 表 | seq + type TEXT + content JSONB | Multica `server/migrations/026_task_messages.up.sql` |
| `workflow_checkpoint` 表 | **LangGraph** `checkpoints` + `checkpoint_writes` | shallow（最新 per run+ns）+ parent_checkpoint_id + state JSONB + pending writes | 研究 R2 |
| `workflow_human_input_task` 表 | **NocoBase** `workflowManualTasks` + Multica `issue` 表 | status CAS + assignee_id + result JSONB + issue_id FK | 研究 R4 |
| `workflow_node_event` 表 | **Multica** `task_message` 表 | seq + type TEXT（无 CHECK）+ content JSONB | Multica `server/migrations/026_task_messages.up.sql` |
| `workflow_env_var` 表 | **Dify** + Multica `plugin_secret` | key + encrypted value + definition_id FK | Dify + Multica `server/migrations/344_plugin_v2_reset.up.sql` |
| 执行引擎 | **LangGraph** Pregel `_loop.py` + `_algo.py` | super-step 遍历 + ready set 计算 + goroutine 并发 + reducer 合并 + checkpoint | 研究 R2 |
| 变量池 | **Dify** `VariablePool` + `VARIABLE_PATTERN` 正则 | `{{#node_id.var#}}` 解析 + sys.* + env.* + Segment 类型化 | Dify `api/core/workflow/` |
| Agent-Call 节点 | **Multica 现有** `CreateAgentTask` + `ClaimAgentTask` + `CreateRetryTask` | 直接调用，复用全部现有机制 | Multica `server/internal/service/task.go` |
| Human-Input CAS | **NocoBase** lockManager + CAS pattern | DB-only CAS `UPDATE...WHERE status='pending' RETURNING *` | 研究 R4 |
| Autopilot 触发器扩展 | **Multica 现有** `autopilot_trigger` + `DispatchAutopilotForPlan` | extend kind='workflow' + autopilot_run.workflow_run_id FK | Multica `server/internal/service/autopilot.go` |
| API handler | **Multica 现有** `autopilot.go` handler | CRUD + trigger REST pattern | Multica `server/internal/handler/autopilot.go` |
| sqlc queries | **Multica 现有** `agent.sql` + `autopilot.sql` 模式 | `-- name: CreateWorkflow :one` 等 sqlc 注释模式 | Multica `server/pkg/db/queries/` |
| testutil fixtures | **Multica 现有** `dbfx` | `dbfx.Workflow`, `dbfx.WorkflowRun` 等 | Multica `server/internal/testutil/db.go` |

## 要做成什么样子

一个**工作流**是一个 JSON DAG（有向无环图），由节点（node）和边（edge）组成，存储为 JSONB blob。工作流被触发后产生一次**工作流执行**（workflow run），按 super-step 模型依次或并行执行节点。

### 节点类型目录

| 节点 | 层 | 作用 | 包装的现有原语 | v1 | v2 |
|---|---|---|---|---|---|
| **Agent-Call** | 执行层 | 调用一个 Agent 执行任务 | `CreateAgentTask` + `ClaimAgentTask` | ✅ | |
| **Squad-Call** | 执行层 | 调用 Squad 多人讨论，输出结构化决策 | Squad leader 路由 + @-mention | | ✅ |
| **Code** | 执行层 | 执行代码（sandbox） | 新建（daemon subprocess） | | ✅ |
| **HTTP Request** | 执行层 | 发起 HTTP 请求 | 新建（net/http） | | ✅ |
| **Human-Input** | 执行层 | 暂停等待人工输入 | 新建（review Issue + Inbox） | ✅ | |
| **IF/ELSE** | 编排层 | 条件分支 | 新建（表达式求值） | ✅ | |
| **Retry-On-Fail** | 编排层 | 失败分支重试（可配 `max_retries`，默认 3） | `CreateRetryTask`（Agent 节点复用） | | ✅ |
| **Variable-Assigner** | 编排层 | 赋值/修改变量池 | 新建 | | ✅ |
| **Sub-Workflow** | 编排层 | 调用已发布的另一个工作流 | 新建 | ✅ | |
| **Output** | 终端层 | 声明工作流最终输出，持久化为 `workflow_run.result` | 新建 | ✅ | |

> **Loop 节点重定义**：原 spec 的 "Iteration / Loop" 重定义为 "Retry-On-Fail"——不是通用循环，而是**失败重试门禁**。只有失败的分支才会进入 Retry-On-Fail 节点。重试上限可配（默认 3），重试耗尽走错误处理分支或 `workflow_run.status = 'failed'`。Agent-Call 节点的重试依赖现有 `CreateRetryTask`（`max_attempts` 字段）；Retry-On-Fail 节点用于非 Agent 节点或整个分支的重试。

### Squad-Call 节点关键设计（v2）

- **终止条件**（可组合，borrow AutoGen `TerminationCondition`）：11 种可组合条件（MaxMessage / TextMention / Timeout / TokenUsage / Handoff / External / FunctionCall / SourceMatch 等），通过 `&`（AND）/ `|`（OR）组合。作为 Squad-Call 节点参数声明，由 Squad-Call runner 在每轮消息 delta 上评估。
- **卡顿检测**（borrow MagenticOne `_n_stalls` / `_max_stalls`）：Squad-Call runner 维护独立 stall 计数器。每轮要求 leader 发射 `LedgerEntry` 风格的结构化评估。progress==False → stall+1，in_loop==True → stall+1，否则 stall-1（衰减）。stall ≥ max_stalls（默认 3）→ 重新规划。卡顿检测是编排逻辑，不是终止条件。
- **结构化决策输出**（`output_schema`）：每个 Squad-Call 节点可声明可配置的 Zod `output_schema`。默认 schema = 简化 LedgerEntry。
- **人工升级**：可配 `escalation_policy: "none" | "human" | "fail"`（默认 "fail"）。
- **结构化 briefing 路径**：Squad-Call 节点注入自己的结构化 briefing **旁路**现有 `buildSquadLeaderBriefing`（非破坏性）。

### 变量池与数据流

节点间通过**变量池**（Variable Pool）传递类型化数据。

- **引用语法**：`{{#node_id.var_name#}}`（Dify 实际语法，`#` 分隔符，正则可解析，避免与 Jinja2/JS 模板冲突）
- **多跳引用**：变量引用支持引用**任意上游节点**的输出，不只是前一个。变量池在每个 super-step 边界累积所有已完成节点的输出。
- **系统变量**：`{{#sys.workflow_run_id#}}` / `{{#sys.timestamp#}}` / `{{#sys.user_id#}}` / `{{#sys.workspace_id#}}`
- **环境变量**：`{{#env.var_name#}}`，存储在独立的 `workflow_env_var` 表中（密钥不随 DSL 导出泄露）
- **节点输入配置**：每个节点在配置面板中**显式声明**输入变量引用（从变量池中选择上游节点输出），而非自动传递前一个节点的输出。
- **Output 节点**：工作流必须有一个 Output 节点声明最终输出。Output 节点从变量池引用 `{{#node_id.var#}}`，其输出持久化为 `workflow_run.result`。
- **Agent Issue 状态暴露**：Agent-Call 节点完成后，引擎在变量池中暴露 `{{#agent_call_n.issue_status#}}`（agent 绑定 Issue 的最终状态）。工作流设计者可通过 IF/ELSE 节点检查此值（如 `== "blocked"` → 走人工介入分支）。引擎不自动暂停——由设计者决定如何处理。

> **语法修正**：原 spec 写的 `{{node_id.output_name}}` 有误——与 Jinja2/JS 模板冲突。改为 Dify 实际语法 `{{#node_id.var_name#}}`。

### Agent 提示词包裹机制

当 Agent-Call 节点声明了 `output_schema`（Zod），引擎自动在 agent 的原始系统提示词基础上**包裹一层**输出提取指令：

```
原始 Agent 系统提示词（用户配置）:
  "你是一个代码审查专家，专注于安全漏洞..."

引擎包裹层（自动注入，追加到 trigger_summary）:
  "完成上述任务后，请按以下 JSON 格式输出结果:
   { "summary": "...", "files_changed": [...], "recommendation": "approve|reject" }
   只输出 JSON，不要其他内容。"
```

- **包裹位置**：追加到 `trigger_summary`（现有 `CreateAgentTask` 字段），不修改 agent 的原始系统提示词
- **提取方式**：从 agent 输出文本中提取 JSON（正则解析 + Zod 验证）
- **失败处理**：提取失败 → 重试最多 3 次；3 次后仍失败 → 节点 failed
- **无 output_schema 时**：不包裹，agent 自由输出（原始文本作为 output）

### 静态模板编辑器

工作流的**定义阶段**需要一个可视化编辑器（本规格包含静态模板 UI）：

- **画布**：@xyflow/react（React Flow v12，当前维护包名）
- **包位置**：`packages/views/workflows/`（web/desktop 共享）
- **路由**：`[workspaceSlug]/(dashboard)/workflows/`（列表）、`workflows/[id]/`（编辑器）、`workflows/new/`（创建）
- **节点面板**：从左侧拖拽节点类型到画布
- **节点配置面板**：点击节点弹出右侧配置（参数表单 + 输入/输出 schema + 变量选择器）
- **边连接**：从节点输出端口拖到下一个节点输入端口。边数据结构为 `edges[]` 数组：`[{id, source, target, sourceHandle, targetHandle, type}]`（React Flow 原生格式）
- **变量选择器**：在配置字段中引用上游节点输出（下拉选择 `{{#node_id.var_name#}}`）
- **Draft 自动保存**：debounced（2s）PUT 整个 graph JSON 到 `PUT /api/workflows/{id}/draft`
- **发布**：Draft → Published（不可变快照，版本化）。同表多行：`version="draft"` vs `version=<timestamp>`

运行时的可视化（节点实时状态、变量检查等）在 [04-可视化运行进展](./04-visual-run-progress.md) 中定义。

> **关于"小队 + 工作流 + 多 Agent 组合"**：这不是独立概念——组合本身就是包含 Squad-Call 节点的 workflow。最复杂的场景（squad 讨论 → workflow 编码 → finalize）= 一个 workflow，里面有三个大节点：Squad-Call（讨论）→ Sub-Workflow（编码 DAG）→ Squad-Call（收尾）。发布出来就是一个 App（见 [02-App 管理](./02-app-management.md)）。

## 借鉴开源组件的哪些能力

> 研究基础：8 份源码分析报告（`.wayfinder/research/R1-R8*.md`），基于 Dify/n8n/NocoBase/LangGraph/AutoGen/MetaGPT/OpenAI Agents SDK 的实际源码（非二手文章）。

### 从 Dify 借鉴（主参考）

| 能力 | 借鉴点 |
|---|---|
| Workflow DSL as JSON DAG | `nodes` + `edges` + `{{#var#}}` 引用的 JSON 结构；存为 JSONB blob（单列 `graph`） |
| `VariablePool` + `sys.*` 系统变量 | `sys.workflow_run_id` / `sys.timestamp` / `sys.user_id` / `sys.workspace_id` |
| Env 变量独立于 DSL | 密钥存 `workflow_env_var` 表，不随 DSL 导出泄露 |
| `ToolParameter.form`（schema/form/llm） | **三层作者权分离**：skill 作者固定（schema）、工作流设计者配置（form）、agent 运行时决定（llm）。是简单枚举（非继承链），直接可适配。增加 Multica 扩展：`issue_binding` 让 llm-form 参数从 Issue 字段直接取值 |
| Draft → Published 快照 | 同表多行：`version="draft"` vs `version=<timestamp>`。运行时 `workflow_run.graph` 复制图快照 |
| `{{#node_id.var_name#}}` 变量语法 | `#` 分隔符，正则可解析，安全（只引用无代码执行） |
| 前端编辑器 UX | React Flow 画布 + 节点面板 + 配置面板 + 变量选择器——技术栈完全匹配，可直接参考 |

### 从 n8n 借鉴（次参考）

| 能力 | 借鉴点 |
|---|---|
| `execution_entity` DB schema | `workflow_run` 表设计参考：status / mode / deduplicationKey（幂等执行） |
| Wait node 暂停/恢复 | `waitTill` sentinel + resume 机制 |
| Execute Workflow node | 子工作流节点：数据在父/子间**复制**（非共享），`parentExecution` 反向引用仅用于元数据/lineage |
| `EXECUTIONS_DATA_SAVE_ON_SUCCESS/ERROR` + prune | 执行数据保留策略：两阶段 soft/hard delete |

### 从 NocoBase 借鉴（次参考）

| 能力 | 借鉴点 |
|---|---|
| `EXECUTION_STATUS` / `JOB_STATUS` 常量 | 完整状态机参考。Multica 采用简化版：`pending → running → waiting → completed/failed/canceled` |
| Manual 节点 resume 模式 | **Human-Input 并发安全参考**。NocoBase 的 in-memory lock 非分布式——Multica 采用 **DB-only CAS**（`UPDATE...WHERE status='pending' RETURNING *`），更健壮，无需 lock manager，跨实例安全 |
| 三种审批模式（single/all/any） | 参考但 v1 仅支持 single assignee |
| DIY/插件扩展模型 | R8 研究中——评估 NocoBase 的"万物皆插件"模式是否对 Spec 03 有额外优势 |

### 从 LangGraph 借鉴（次参考——引擎核心）

| 能力 | 借鉴点 |
|---|---|
| `StateGraph` + checkpointer | 工作流执行的 case state 基质。Multica 采用 **ShallowPostgresSaver 模式**（仅保留最新 checkpoint per run+namespace），schema 设计支持未来全历史（time-travel 调试） |
| `interrupt()` + `Command(resume=)` | Human-Input 节点的暂停/恢复原语。interrupt → checkpoint + pending writes → resume 写入 RESUME → 节点重新执行 |
| `Send` + reducer | 并行 fan-out：一个节点发射多个 `Send(node, state)`，结果通过 reducer 合并。**这是唯一有正确并发写合并的模型**——Dify 和 n8n 都没有 reducer 概念 |
| Subgraph checkpoint namespace | 子工作流隔离。Multica 简化为：子工作流 = 新 `workflow_run` 行 + `parent_workflow_run_id` FK |
| Pregel super-step 模型 | 执行引擎并发模型：所有 ready 节点并行 goroutine → reducer 合并 → checkpoint → 下一 super-step |

### 从 AutoGen 借鉴（次参考——Squad-Call 专用）

| 能力 | 借鉴点 |
|---|---|
| 可组合 `TerminationCondition`（11 种） | Squad-Call 节点的终止条件参数。通过 `&`（AND）/ `|`（OR）组合 |
| `MagenticOneOrchestrator` stall 检测 | `_n_stalls` / `_max_stalls`（默认 3）。stall 检测是编排逻辑，不是终止条件 |
| `LedgerEntry` 结构化评估 | 5 字段 schema，每字段 `{reason, answer}`。SquadDecision 默认 schema 的基础 |

### 从 OpenAI Agents SDK 借鉴（次参考——节点契约）

| 能力 | 借鉴点 |
|---|---|
| `handoff(input_type=)` + `Agent(output_type=)` | 节点类型化交接契约：每个节点声明 Zod `inputSchema` / `outputSchema` |
| `AgentOutputSchema.validate_json()` | 节点输出验证：Zod schema 验证 + 重试 |
| `InputGuardrail` / `OutputGuardrail` | 节点级策略验证（pre/post-conditions），tripwire-on-fail |
| `with trace("case-id")` | 一次工作流执行 = 一个可审计 trace，所有节点执行归入一个单元 |

## 与 Multica 现有原语的关系

| Multica 原语 | 工作流中的角色 |
|---|---|
| `agent_task_queue`（Run，70+ 列） | Agent-Call 节点调用 `CreateAgentTask`，产生一个 Run。引擎不运行 agent——只创建 task 行，daemon 认领执行 |
| `ClaimAgentTask`（FOR UPDATE SKIP LOCKED） | daemon 认领机制。per-(issue,agent) 序列化确保同一 issue+agent 无并发任务 |
| `CreateRetryTask`（clone + ON CONFLICT DO NOTHING） | Agent-Call 节点的重试 chaining。attempt+1，clone parent，幂等 |
| `task_message`（type = free TEXT，无 CHECK） | Agent-Call 节点的执行日志。**工作流节点事件使用独立的 `workflow_node_event` 表**，不扩展 task_message（避免 FK 耦合） |
| `TaskProgressPayload{Step, Total}` | 已有 step/total wire format。工作流进度复用此模式 |
| `Autopilot` triggers（schedule/webhook/api） | 工作流触发器**复用** Autopilot 机制：扩展 `autopilot_trigger.kind` 为 'workflow'，`autopilot_run` 增加 `workflow_run_id` FK |
| `issue.stage` + `stageBarrierClosed` | **完全不耦合**——migration 123 明确说"server 没有声明式工作流模型"。工作流引擎是声明式层，不重载 issue.stage |
| `Skill`（SKILL.md + skill_file） | 工作流节点可引用 Skill。ToolParameter.form 三层作者权扩展 `skill.config` 的 `parameters` 数组 |
| `Squad`（leader 路由 + @-mention） | Squad-Call 节点包装现有 leader 路由 + @-mention 委派。结构化 briefing **旁路**现有 `buildSquadLeaderBriefing`（非破坏性） |
| `Plugin Contributes`（Surfaces/Hooks/Resources） | 新增 `workflow` 作为 Resource 类型（扩展路径明确：新 const + HostCapabilities flip + manifest 验证 + install-time writer） |

## 范围边界

**本规格包含**：
- 工作流定义模型（DAG JSON 结构 + 变量池 + 节点类型目录）
- 执行引擎（super-step 遍历 + 串/并/分支/暂停恢复 + checkpoint）
- 静态模板编辑器（@xyflow/react 画布 + 节点配置面板 + draft 自动保存 + 发布）
- 变量传递机制（`{{#node_id.var#}}` 引用 + `sys.*` + env vars + 多跳引用 + 显式 Output 节点）
- Agent 提示词包裹机制（output_schema → 自动注入输出提取指令）
- Human-Input 节点（暂停 → 创建 review Issue → DB-only CAS 并发安全 → 恢复）
- Sub-Workflow 节点（子工作流调用 + namespace 隔离 + 数据复制非共享）

**本规格不包含**（在其他规格中或延后）：
- Chatflow 模式（围绕 Issue 评论的多轮对话工作流）——延后到 v2
- Squad-Call 节点实现——v2
- Code 节点沙箱——v2
- HTTP Request 节点——v2
- Retry-On-Fail 节点——v2
- Variable-Assigner 节点——v2
- 运行时可视化（节点实时状态、变量检查器）→ [04-可视化运行进展](./04-visual-run-progress.md)
- App 发布面（将工作流发布为可调用工具）→ [02-App 管理](./02-app-management.md)
- 插件贡献自定义节点类型 → [03-DIY 插件扩展](./03-diy-plugin-extensibility.md)

## User Stories

1. As a workspace admin, I want to create a workflow by dragging nodes onto a canvas and connecting them with edges, so that I can visually define a deterministic multi-step process.
2. As a workspace admin, I want to configure each node's input variables by selecting from upstream node outputs in a dropdown, so that data flows correctly between non-adjacent nodes.
3. As a workspace admin, I want to declare an Output node that references specific upstream outputs, so that the workflow's final result is well-defined and queryable.
4. As a workspace admin, I want to publish a draft workflow as an immutable snapshot, so that triggered runs always execute against a frozen definition.
5. As a workspace admin, I want to trigger a published workflow via API call, so that I can integrate it into external systems.
6. As a workspace admin, I want to trigger a published workflow via Autopilot cron schedule, so that recurring tasks run automatically.
7. As a workspace admin, I want to trigger a published workflow via webhook, so that external events can start a workflow.
8. As an agent, I want an Agent-Call node to dispatch me via the existing agent_task_queue, so that my execution reuses all existing daemon/runtime/retry infrastructure.
9. As an agent, I want the workflow engine to optionally wrap my system prompt with output extraction instructions, so that my output is structured JSON when the downstream node needs typed data.
10. As a workflow designer, I want to use IF/ELSE nodes to branch based on upstream node outputs (including agent issue_status), so that the workflow adapts to runtime conditions.
11. As a human reviewer, I want a Human-Input node to create a review Issue visible on the board, so that I can see and respond to pending workflow approvals.
12. As a human reviewer, I want the workflow to pause until I submit my input, so that the process doesn't continue without my decision.
13. As a human reviewer, I want concurrent submissions to the same Human-Input task to be safely serialized (only first wins), so that double-submits don't corrupt state.
14. As a workflow designer, I want to use Sub-Workflow nodes to call reusable sub-processes, so that complex workflows compose from simpler ones.
15. As a workspace admin, I want to see workflow run status (pending/running/waiting/completed/failed/canceled), so that I can monitor execution.
16. As a workspace admin, I want to see per-node execution status and token usage, so that I can audit and optimize workflows.
17. As a workspace admin, I want failed workflow runs to be resumable from the last checkpoint, so that transient failures don't require restarting from scratch.
18. As a developer, I want each node type to declare Zod input/output schemas, so that data contracts are enforced at node boundaries.
19. As a plugin author, I want to contribute a workflow definition as a plugin Resource, so that reusable workflows can be shared and installed.
20. As a workflow designer, I want to check an agent's issue_status after Agent-Call completes, so that I can route to a human intervention branch if the agent marked the issue as blocked.

## Implementation Decisions

### 引擎架构

- **引擎位置**：新建 Go 模块 `server/internal/workflow/`。引擎是 server 进程内的 Go 代码，编译 JSON DAG → 运行时图 → super-step 遍历。Daemon 不变——只知道 `agent_task_queue` 行。
- **并发模型**：super-step（LangGraph Pregel 风格）。引擎计算 "ready set"（上游依赖全完成的节点），goroutine 并行执行，reducer 合并 fan-in 结果，checkpoint，计算下一 ready set。
- **Agent 调度**：Agent-Call 节点调用 `CreateAgentTask`（同进程 Go 调用），创建 `agent_task_queue` 行（status='queued'）。引擎记录 task_id 到 `workflow_node_execution` 行，不阻塞。Daemon 认领并执行 agent CLI。引擎通过事件总线（`task:completed`）或轮询检测完成，读取 agent task result，写入变量池，进入下一 super-step。
- **Agent Issue 状态暴露**：Agent-Call 节点完成后，引擎读取绑定 Issue 的状态，写入变量池 `{{#agent_call_n.issue_status#}}`。引擎不自动暂停——由工作流设计者通过 IF/ELSE 检查决定后续行为。
- **错误传播**：Agent-Call 节点失败 → `CreateRetryTask`（复用现有机制，attempt+1）。非 Agent 节点失败 → Retry-On-Fail 节点（v2）。重试耗尽 → `workflow_run.status = 'failed'`。

### 数据库 Schema

- `workflow_definition`：`id, workspace_id, name, description, graph JSONB, version TEXT (draft|<timestamp>), environment_variables JSONB (separate from graph), created_by, created_at, updated_at`
- `workflow_run`：`id, workflow_definition_id, workspace_id, issue_id (nullable), status (pending|running|waiting|completed|failed|canceled), trigger_source (schedule|webhook|api|manual), trigger_payload JSONB, result JSONB, started_at, completed_at, created_at`
- `workflow_node_execution`：`id, workflow_run_id, node_id (references node in graph JSON), node_type, status (pending|running|waiting|completed|failed), input JSONB, output JSONB, agent_task_id (nullable FK → agent_task_queue, for Agent-Call nodes), started_at, completed_at, created_at`
- `workflow_checkpoint`：`id, workflow_run_id, namespace TEXT, parent_checkpoint_id (nullable, for future history), state JSONB, created_at`。MVP: shallow（仅保留最新 per run+namespace）
- `workflow_checkpoint_write`：`id, workflow_run_id, checkpoint_id, type (interrupt|resume), payload JSONB, created_at`
- `workflow_human_input_task`：`id, workflow_run_id, workflow_node_execution_id, issue_id (review Issue), assignee_id, status (pending|resolved|cancelled), result JSONB, created_at, resolved_at`
- `workflow_node_event`：`id, workflow_run_id, workflow_node_execution_id, seq INT, type TEXT (node_started|node_completed|node_failed|workflow_started|workflow_completed), content JSONB, created_at`
- `workflow_env_var`：`id, workflow_definition_id, key TEXT, value TEXT (encrypted for secrets), created_at`

### API 面

- REST：`GET/POST/PUT/DELETE /api/workflows`（CRUD）、`GET/PUT /api/workflows/{id}/draft`（draft graph）、`POST /api/workflows/{id}/publish`（draft→published）、`POST /api/workflows/{id}/trigger`（手动触发）、`GET /api/workflows/runs`（run 列表）、`GET /api/workflows/runs/{id}`（run 详情）、`GET /api/workflows/runs/{id}/events?since=N`（事件日志）
- WebSocket：新事件类型 `workflow:started`、`workflow:node_started`、`workflow:node_completed`、`workflow:node_failed`、`workflow:completed`、`workflow:failed`、`workflow:waiting`
- Auth：workspace-scoped + RBAC（owner/admin 创建/编辑/发布；member 触发+查看）
- 并发限制：per-workflow-definition max concurrent runs（类似 Autopilot `concurrency_policy`）

### 触发机制

- 复用 Autopilot：`autopilot_trigger.kind` 扩展为 'workflow'；`autopilot_run` 增加 `workflow_run_id` FK。`execution_mode` 新增 'workflow'。
- 手动触发：`POST /api/workflows/{id}/trigger`

### 节点类型契约

- 每个节点类型声明 Zod `inputSchema` / `outputSchema` + `execute(input, ctx) → output`
- 可选 guardrails（pre/post-conditions，borrow OpenAI SDK InputGuardrail/OutputGuardrail）
- `NodeRegistry` 模式：内置类型启动时注册；Spec 03 可扩展插件类型
- v1 节点是纯函数：`execute(input) → output`。引擎处理生命周期（retry、checkpoint、事件）。

### ToolParameter.form 三层作者权

- 采用 Dify 三层枚举：`schema`（skill 作者固定）、`form`（工作流设计者配置）、`llm`（agent 运行时决定）
- Multica 扩展：`issue_binding` 让 llm-form 参数从 Issue 字段直接取值
- 参数存储：`skill.config` JSONB 的 `parameters` 数组
- 三级优先级：`workflow node config > agent runtime config > skill schema default`

### 编辑器

- `@xyflow/react` v12
- 包位置：`packages/views/workflows/`
- 路由：`[workspaceSlug]/(dashboard)/workflows/`
- 全 spec 功能：画布 + 节点面板 + 配置面板 + 边连接 + 变量选择器 + draft 自动保存 + 发布

## Testing Decisions

- **测试策略**：特性级（跨多类多函数）功能测试为主，单元级 table-driven 为辅。不使用 Cucumber（避免 JVM + Gherkin 复杂度），用 Go test 场景化测试实现 BDD 风格。
- **引擎 super-step 遍历**：测试 DAG 拓扑解析 + ready set 计算 + reducer 合并。不测 Go goroutine 内部，只测 super-step 输出（哪些节点执行了、变量池最终状态）。
- **Agent-Call 调度**：mock `CreateAgentTask`，验证 task 创建参数正确（agent_id, issue_id, trigger_summary 含包裹提示词）。不启动真实 daemon。
- **Agent Issue 状态暴露**：mock agent task 完成后 Issue 状态为 blocked，验证 `{{#agent_call_n.issue_status#}}` 写入变量池，IF/ELSE 节点正确路由。
- **Human-Input CAS**：并发提交测试——两个 goroutine 同时 CAS 更新同一 task，验证只有一个成功（0 rows affected → 409）。
- **Checkpoint 恢复**：创建 checkpoint → 模拟中断 → 从 checkpoint 恢复 → 验证状态一致。
- **变量池引用解析**：`{{#node_id.var_name#}}` 正则解析 + 多跳引用 + sys.* + env.* + 缺失引用错误。
- **编辑器**：React Flow 组件渲染 + drag-drop + 边连接 + 配置面板。使用 `@testing-library/react` + `@xyflow/react` test utils。
- **DB schema**：sqlc 查询测试 + migration up/down 测试。复用 `server/internal/testutil` fixtures。
- **API**：`testutil.Call(h, req).Want(status).JSON(&out)` 模式（复用现有 Go handler 测试）。
- **Event bus**：验证 `workflow:node_completed` 事件在节点完成后被发布。

## Out of Scope

- Chatflow 模式（围绕 Issue 评论的多轮对话工作流）——v2
- Squad-Call 节点实现——v2
- Code 节点沙箱——v2
- HTTP Request 节点——v2
- Retry-On-Fail 节点——v2
- Variable-Assigner 节点——v2
- App 发布面（Spec 02）
- 运行时可视化（Spec 04）——静态模板编辑器在 Spec 01
- 插件自定义节点类型实现（Spec 03）——但节点类型契约在 Spec 01
- Mobile 端工作流编辑器
- 重载 issue.stage 为声明式 stage 模型

## Further Notes

- **v1 MVP 节点类型**：Agent-Call、IF/ELSE、Sub-Workflow、Human-Input、Output（5 种）。Squad-Call、Code、HTTP、Retry-On-Fail、Variable-Assigner 延后到 v2。
- **v1 只有 Workflow 模式**（无状态，触发→执行→输出）。Chatflow 模式延后到 v2。
- **Spec 01 语法修正**：原 `{{node_id.output_name}}` 改为 `{{#node_id.var_name#}}`（Dify 实际语法，`#` 分隔符）。
- **Agent 提示词包裹**：Agent-Call 节点声明 `output_schema` 时自动包裹。无 output_schema = 自由输出。包裹层追加到 `trigger_summary`，不修改 agent 原始系统提示词。
- **Loop 重定义**：原 "Iteration / Loop" 重定义为 "Retry-On-Fail"——失败重试门禁，非通用循环。
- **Human-Input 并发安全**：DB-only CAS（无 lock manager），比 NocoBase 的 in-memory mutex 更健壮（跨实例安全）。
- **issue.stage 完全独立**：工作流引擎不与 `stageBarrierClosed` 交互。DAG 拓扑本身是声明式模型。
- **Agent Issue 状态**：引擎不自动暂停。变量池暴露 `issue_status`，设计者用 IF/ELSE 决定后续行为。
- **主参考 Dify**：前端编辑器 + DAG 数据结构 + 变量池 + 节点参数模型直接参考 Dify（React + React Flow 技术栈匹配）。后端引擎参考 LangGraph（Pregel super-step）。NocoBase 的 DIY/插件扩展性待 R8 研究确认是否对 Spec 03 有额外优势。

## 验收标准

1. 用户可以在 @xyflow/react 画布上拖拽节点、连接边、配置参数（含变量选择器），保存为 draft
2. Draft 可以发布为不可变快照（draft → published）
3. 已发布的工作流可通过 API / webhook / Autopilot cron / 手动触发
4. 执行时按 super-step 模型依次/并行执行节点
5. Agent-Call 节点调用 `CreateAgentTask` 创建 `agent_task_queue` 行，daemon 认领执行真实 agent CLI
6. Agent-Call 节点声明 output_schema 时，agent 提示词被自动包裹输出提取指令
7. 节点间变量通过 `{{#node_id.var_name#}}` 正确传递，支持多跳引用
8. Output 节点声明工作流最终输出，持久化为 `workflow_run.result`
9. Human-Input 节点暂停执行、创建 review Issue、Inbox 通知、人工提交后恢复
10. Human-Input 并发提交安全（CAS，只有第一个提交成功）
11. 并行 fan-out 分支正确执行，结果通过 reducer 正确合并
12. 失败节点可重试（Agent-Call 复用 `CreateRetryTask`）
13. 失败的工作流可从 checkpoint 恢复
14. 一次工作流执行的全部节点状态、输入输出、耗时、token 消耗可审计（`workflow_node_event` 表）
15. Sub-Workflow 节点调用已发布的另一个工作流，数据在父/子间复制（非共享），namespace 隔离
16. Agent-Call 节点完成后，`{{#agent_call_n.issue_status#}}` 暴露在变量池中，IF/ELSE 节点可检查此值路由分支
