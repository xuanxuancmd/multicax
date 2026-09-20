# Multica Workflow Engine — Context Glossary

> Domain glossary for the Workflow Engine (Spec 01). Implementation-agnostic.
> Created during G1-G4 grilling sessions.

## Core Entities

### Workflow Definition
The published, immutable JSON DAG (nodes + edges + variable references). Has a Draft (editable) and Published (immutable snapshot) version. Stored as a JSONB blob. Not to be confused with a Workflow Run.

### Workflow Run
One execution of a published Workflow Definition. Has a status (pending → running → waiting → completed/failed/canceled). May or may not be bound to an Issue (depends on mode). Contains one or more Workflow Node Executions.

### Workflow Node Execution
One node's execution within a Workflow Run. Has its own status (pending → running → waiting → completed/failed). For Agent-Call nodes, references an `agent_task_queue` row. For other node types, execution is self-contained.

### Agent Task Queue (existing)
Multica's existing primitive for agent execution. 70+ columns of agent runtime state (runtime_id, session_id, work_dir, etc.). Agent-Call workflow nodes delegate to this via `CreateAgentTask`, but workflow node execution is a separate concept.

### Variable Pool
Runtime storage for typed variables passed between nodes. Uses `{{#node_id.var_name#}}` reference syntax. Contains `sys.*` system variables, `env.*` environment variables, and node outputs.

## Node Types

### Agent-Call
Calls an Agent via `CreateAgentTask`. Waits for the agent task to complete. Output = agent task result.

### Squad-Call
Calls a Squad via existing leader routing. Has composable termination conditions, stall detection, and structured decision output (SquadDecision). Output = SquadDecision.

### Human-Input
Pauses workflow execution. Creates a review Issue. Notifies via Inbox. Resumes on human submission. Uses DB-only CAS for concurrency safety (no lock manager).

### Sub-Workflow
Calls another published workflow. Data is copied (not shared) between parent and child. Uses checkpoint namespace for isolation.

## Execution Model

### Super-Step
The execution engine computes the "ready set" (nodes whose upstream dependencies are complete), executes them as goroutines, merges results via reducers, checkpoints, then computes the next ready set.

### Checkpoint
Persisted state of a Workflow Run at a super-step boundary. MVP uses shallow checkpointing (latest per run). Schema supports full history (for time-travel debugging) as a future extension.

### Interrupt / Resume
Human-Input and Sub-Workflow nodes pause execution. The checkpoint stores the interrupt state. Resume writes a RESUME entry, and the interrupted node re-executes with the resume value available.

## Modes

### Workflow Mode
Stateless: trigger → execute → output. No Issue binding. Each run is independent.

### Chatflow Mode
Stateful: multi-turn around Issue comments. Bound to an Issue. Variable pool persists across turns.
