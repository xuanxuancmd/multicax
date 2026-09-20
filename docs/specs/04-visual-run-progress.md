# 04 — 可视化运行进展

> **全局目标**：AI 辅助研发平台，支持 DIY 自定义 App 发布可复用的能力，后台对接 Claude CLI（现有 Multica 机制）。
> 本规格定义运行时的可视化：DAG 画布实时状态 + 节点级变量检查 + 小队讨论线程 + 执行时间线，UI 足够美观。

## 要做成什么样子

### 场景一：工作流运行时 DAG 画布

工作流执行时，用户在画布上看到**实时的节点状态变化**：

```
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  Start   │────►│  Plan    │────►│ Research │
  │ ✓ Done   │     │ ✓ Done   │     │ ✓ Done   │
  └──────────┘     └──────────┘     └─────┬────┘
                                          │
                          ┌───────────────┼───────────────┐
                          ▼               ▼               ▼
                   ┌──────────┐    ┌──────────┐    ┌──────────┐
                   │ Code: A  │    │ Code: B  │    │ Code: C  │
                   │ ✓ Done   │    │ ● Running│    │ ○ Pending│
                   └─────┬────┘    └─────┬────┘    └─────┬────┘
                         │               │               │
                         └───────────────┼───────────────┘
                                         ▼
                                   ┌──────────┐
                                   │  Test    │
                                   │ ○ Waiting│
                                   └──────────┘
```

- **节点状态颜色**：✓ 绿色（Done）/ ● 蓝色脉冲（Running）/ ○ 灰色（Pending）/ ✗ 红色（Failed）/ ⏸ 黄色（Paused）
- **边动画**：数据流动时边有流动动画；正在执行的连接高亮
- **画布只读**：执行中画布锁定不可编辑（`canvasReadOnly`）
- **节点点击**：点击节点弹出侧栏面板，显示该节点的 inputs / process_data / outputs / elapsed / tokens
- **streaming tokens**：Agent 节点执行时，侧栏面板实时流式显示 LLM 输出 tokens
- **失败 UX**：节点失败时画布上标红，自动切到 DETAIL tab，failed 节点展开显示错误信息

### 场景二：三面板调试 UX

一次工作流执行有三个调试视角（Tab 切换）：

| Tab | 内容 | 借鉴 |
|---|---|---|
| **RESULT** | 最终输出（workflow_run.output 的结构化展示） | Dify RunPanel RESULT tab |
| **DETAIL** | 执行追踪时间线：每个节点一行，显示 node_name / status / elapsed / tokens，点击展开 inputs/outputs | Dify RunPanel DETAIL tab |
| **TRACING** | 完整事件流：按时间顺序列出所有 node_started / node_finished / text_chunk / error 事件（WebSocket 实时流） | Dify RunPanel TRACING tab + n8n executions view |

### 场景三：小队讨论线程可视化

小队讨论不是 DAG，而是**多轮对话线程**。需要在 Issue timeline 中渲染：

```
  ┌─ Issue: "实现用户注册功能" ──────────────────────────┐
  │                                                      │
  │  👤 张三 (reporter)                                  │
  │  └─ 需求：实现用户注册功能，需要邮箱验证              │
  │                                                      │
  │  🤖 Squad Leader (路由决策)                          │
  │  ├─ 评估：需要后端 API + 前端表单 + 邮件服务          │
  │  ├─ 路由：@后端Agent 实现注册 API                     │
  │  └─ 路由：@前端Agent 实现注册表单                     │
  │                                                      │
  │  🤖 后端Agent (执行中)                               │
  │  ├─ 正在实现 /api/register endpoint                  │
  │  ├─ [工具调用] POST /api/register                    │
  │  └─ [输出] 注册 API 已实现，返回 user_id + token      │
  │                                                      │
  │  🤖 前端Agent (排队中)                               │
  │  └─ 等待后端 API 完成后开始                           │
  │                                                      │
  │  📊 Squad Leader 状态面板                            │
  │  ├─ 后端Agent: ● Working (已运行 3m 20s)              │
  │  ├─ 前端Agent: ○ Queued                              │
  │  └─ 整体进度: 1/2 完成                               │
  └──────────────────────────────────────────────────────┘
```

- **leader 路由决策**：显示 leader 的评估 + 路由选择（@-mention 了谁）
- **成员执行状态**：每个成员一行，显示 working/idle/offline + 已运行时间
- **工具调用流**：实时流式显示 agent 的工具调用（复用现有 task_message transcript）
- **leader 再触发**：member 完成后 leader 自动再评估，显示 "leader 重新评估中..."

### 场景四：执行时间线（跨工作流 + 小队）

对于一个完整的 Case（squad 讨论 → workflow 编码 → 收尾），需要一个**高层时间线**：

```
  Case: "实现用户注册功能"
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  
  [09:00] ──► [squad 讨论] ──────────────► [10:30]
              leader 路由 + 3 轮讨论
              产出: SquadDecision
                         │
  [10:30] ──► [workflow 编码] ──────────► [13:00]
              Plan → Research → Code(A/B/C 并行) → Test → Review
              产出: CodeArtifact[]
                         │
  [13:00] ──► [finalize 收尾] ──────────► [14:00]
              review → docs → sign-off
              产出: FinalizationReport
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  总耗时: 5h | Token: 2.3M | 产出: 3 files + 2 docs
```

- 每个 phase 可展开查看内部详情（squad 讨论线程 / workflow DAG / finalize 线程）
- 时间线复用现有 execution log 的 WebSocket 流

## 借鉴开源组件的哪些能力

### 从 Dify 借鉴

| 能力 | 借鉴点 |
|---|---|
| React Flow 画布 + `CustomNode` / `CustomEdge` / `_runningStatus` | DAG 画布的节点/边渲染 + 实时状态 |
| `NodePanel`（inputs / process_data / outputs / elapsed / tokens） | 节点点击后的侧栏详情面板 |
| `RunPanel` 三 tab（RESULT / DETAIL / TRACING） | 调试 UX 的三视角 |
| `useWorkflowFailed` hook | 失败 UX：红色标记 + 自动切到 DETAIL tab |
| `canvasReadOnly` | 执行中画布锁定 |
| streaming tokens（`NodeRunStreamChunkEvent` → SSE → 前端追加） | Agent 节点的实时 token 流式显示 |
| `WorkflowDraftSlice` + `WorkflowSlice`（Zustand） | 画布状态管理（draft 自动保存 + running data） |

### 从 n8n 借鉴

| 能力 | 借鉴点 |
|---|---|
| `CanvasNodeDefault.vue` 状态 CSS class（`$style.success` / `error` / `running` / `waiting` / `pinned`） | 节点状态的可视化样式 |
| `useWorkflowDocumentRenderData`（`executionStatusByNodeId` / `executionRunningByNodeId` / `executionWaitingByNodeId`） | 节点级实时状态的数据管道 |
| `RunData.vue`（JSON / Table / Schema / Binary tabs） | 节点输出查看器的多视角 |
| `OutputPanel.vue`（`features/ndv/panel/components/`） | 节点输出面板组件 |
| 全局 Executions 侧栏 + per-workflow Executions tab | 执行历史列表 + 详情查看 |
| `EXECUTIONS_DATA_PRUNE` / `EXECUTIONS_DATA_MAX_AGE` | 执行数据保留策略 |
| Pin data（`$style.pinned` + `dirtiness`） | 测试时固定节点输出 |

### 从 NocoBase 借鉴

| 能力 | 借鉴点 |
|---|---|
| `Instruction` client 注册（`FieldsetLoader` for config form + `ComponentLoader` for canvas rendering） | 工作流节点的配置面板 + 画布渲染分离 |
| `branching` / `end` / `testable` / `useVariables` 指令属性 | 节点的可视化行为声明（是否分支 / 是否终止 / 是否可测试 / 是否使用变量） |
| 执行历史：per-node status + result data + execution plan status | 工作流执行历史的 per-node 展示 |
| 4 个节点分组（control / collection / manual / extended） | 节点面板的分组组织 |

## 与 Multica 现有原语的关系

| Multica 原语 | 可视化中的角色 |
|---|---|
| `task_message`（执行日志，WebSocket 流式） | 工作流节点事件的实时流来源（扩展 `type=node_started/node_finished/text_chunk`） |
| `TaskProgressPayload{Step, Total}` | 已有 step/total wire format，扩展为持久化 step 历史 |
| `packages/views/common/task-transcript/` | 现有 transcript 查看器，扩展为 DAG 画布 + 三面板调试 |
| Board UI（Kanban 看板 + 拖拽） | 现有看板组件，DAG 画布作为新视图类型 |
| `"Engineer is working"` chip | 现有 agent 工作状态指示，扩展为节点级状态 |
| `squad_activity_log` + leader 评估记录 | 小队讨论线程的数据来源 |
| `squad_member_status`（working/idle/offline/unstable） | 小队成员状态面板的数据来源 |
| execution log sidebar（issue 右侧） | 现有执行日志侧栏，扩展为工作流运行面板 |

## 范围边界

**本规格包含**：
- 工作流运行时 DAG 画布（节点实时状态 + 边动画 + 画布只读）
- 节点详情侧栏面板（inputs / process_data / outputs / elapsed / tokens + streaming tokens）
- 三面板调试 UX（RESULT / DETAIL / TRACING）
- 失败 UX（红色标记 + 自动切 tab + 错误展开）
- 小队讨论线程可视化（leader 路由决策 + 成员状态 + 工具调用流）
- Case 高层时间线（squad → workflow → finalize 的跨 phase 时间线）
- 执行历史列表 + 详情查看
- 执行数据保留策略

**本规格不包含**（在其他规格中）：
- 工作流模板的静态编辑器画布 → [01-工作流引擎](./01-workflow-engine.md)
- App 管理界面 → [02-App 管理](./02-app-management.md)
- 插件贡献自定义可视化组件 → [03-DIY 插件扩展](./03-diy-plugin-extensibility.md)

## 验收标准概要

1. 工作流执行时，DAG 画布上节点状态实时变化（绿色 Done / 蓝色 Running / 灰色 Pending / 红色 Failed / 黄色 Paused）
2. 点击节点弹出侧栏，显示该节点的 inputs / process_data / outputs / elapsed / tokens
3. Agent 节点执行时，侧栏实时流式显示 LLM 输出 tokens
4. 三面板调试 UX（RESULT / DETAIL / TRACING）正确切换，数据正确展示
5. 节点失败时画布标红，自动切到 DETAIL tab，failed 节点展开错误信息
6. 小队讨论线程在 Issue timeline 中正确渲染（leader 路由决策 + 成员状态 + 工具调用流）
7. Case 高层时间线正确展示（squad → workflow → finalize 的跨 phase 视图）
8. 每个阶段可展开查看内部详情
9. 执行历史列表 + 详情查看可用
10. 画布 UI 美观（动画流畅、状态颜色清晰、布局合理）
