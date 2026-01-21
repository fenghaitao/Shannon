# Task Lifecycle

<cite>
**Referenced Files in This Document**
- [main.go](file://go/orchestrator/main.go)
- [types.go](file://go/orchestrator/internal/workflows/types.go)
- [models.go](file://go/orchestrator/internal/db/models.go)
- [adapter.go](file://go/orchestrator/internal/temporal/adapter.go)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go)
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go)
- [control_signals.go](file://go/orchestrator/internal/workflows/control_signals.go)
</cite>

## Table of Contents
1. [Introduction](#introduction)
2. [Project Structure](#project-structure)
3. [Core Components](#core-components)
4. [Architecture Overview](#architecture-overview)
5. [Detailed Component Analysis](#detailed-component-analysis)
6. [Dependency Analysis](#dependency-analysis)
7. [Performance Considerations](#performance-considerations)
8. [Troubleshooting Guide](#troubleshooting-guide)
9. [Conclusion](#conclusion)
10. [Appendices](#appendices)

## Introduction
This document describes the complete task lifecycle in the system, from submission to completion. It explains task states, transitions, and checkpoints during workflow execution; temporal workflow integration for deterministic execution; retry mechanisms and failure recovery; control signals for pause/resume/cancel; task metadata, timestamps, and audit trails; practical examples for monitoring progress, handling timeouts, and managing concurrent executions; and guidance for debugging workflow issues and understanding execution statistics.

## Project Structure
The task lifecycle spans several layers:
- Orchestration entrypoint initializes services, workers, and Temporal client.
- Workflows define deterministic execution plans and checkpoints.
- Activities implement atomic units of work.
- Database models capture task execution, agent/tool execution, and audit logs.
- Control signals enable pause/resume/cancel propagation across parent and child workflows.

```mermaid
graph TB
subgraph "Orchestrator"
M["main.go<br/>Service bootstrap, Temporal worker, HTTP/admin endpoints"]
end
subgraph "Workflows"
SW["simple_workflow.go<br/>SimpleTaskWorkflow"]
WU["supervisor_workflow.go<br/>SupervisorWorkflow"]
CTRL["control/handler.go<br/>SignalHandler"]
SIG["control/signals.go<br/>Control signals"]
end
subgraph "Temporal"
TA["adapter.go<br/>ZapAdapter for Temporal logs"]
end
subgraph "Persistence"
DBM["models.go<br/>TaskExecution, AgentExecution, ToolExecution, AuditLog"]
end
M --> SW
M --> WU
M --> TA
SW --> CTRL
WU --> CTRL
CTRL --> SIG
SW --> DBM
WU --> DBM
```

**Diagram sources**
- [main.go](file://go/orchestrator/main.go#L49-L799)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L19-L664)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L40-L1635)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L14-L279)
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go#L5-L41)
- [adapter.go](file://go/orchestrator/internal/temporal/adapter.go#L11-L90)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L219)

**Section sources**
- [main.go](file://go/orchestrator/main.go#L49-L799)

## Core Components
- TaskInput and TaskResult define the shape of task submissions and results.
- TaskExecution, AgentExecution, ToolExecution, and AuditLog represent persisted state and audit trails.
- SimpleTaskWorkflow and SupervisorWorkflow implement deterministic execution with checkpoints and retries.
- SignalHandler coordinates pause/resume/cancel across parent and child workflows.

**Section sources**
- [types.go](file://go/orchestrator/internal/workflows/types.go#L8-L59)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L219)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L19-L664)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L40-L1635)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L14-L279)

## Architecture Overview
The system integrates gRPC, HTTP admin endpoints, Temporal workers, and persistence. Workflows emit structured events for streaming and dashboards, and activities persist execution artifacts.

```mermaid
sequenceDiagram
participant Client as "Client"
participant GRPC as "gRPC Server"
participant Temporal as "Temporal Client"
participant Worker as "Temporal Worker"
participant WF as "Workflow (Supervisor/Simple)"
participant Act as "Activity"
participant DB as "DB Models"
Client->>GRPC : Submit task
GRPC->>Temporal : Start workflow
Temporal->>Worker : Dispatch workflow task
Worker->>WF : Execute workflow
WF->>Act : Execute activities (memory, tool, synthesis)
Act->>DB : Persist AgentExecution/ToolExecution
WF-->>GRPC : Emit events (streaming)
GRPC-->>Client : Streaming updates
WF-->>Temporal : Complete workflow
Temporal-->>GRPC : Finalize
```

**Diagram sources**
- [main.go](file://go/orchestrator/main.go#L396-L799)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L40-L1635)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L19-L664)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L152)

## Detailed Component Analysis

### Task States and Transitions
- Workflow lifecycle events are emitted at key transitions:
  - Workflow started
  - Agent thinking/started/completed
  - Progress updates (subtasks, dependencies, budget)
  - Data processing (compression, synthesis)
  - LLM output (final answer)
  - Workflow completed
  - Pausing/resuming/cancelling (with checkpoints)
- Control signals:
  - Pause: blocks at checkpoints until resumed or cancelled.
  - Resume: clears paused state and continues.
  - Cancel: sets cancelled state and propagates to children.

```mermaid
stateDiagram-v2
[*] --> Started
Started --> Thinking : "emit thinking"
Thinking --> AgentStarted : "execute agent/memory"
AgentStarted --> Executing : "tool/memory/synthesis"
Executing --> Paused : "pause signal"
Paused --> Executing : "resume signal"
Executing --> Completed : "success"
Executing --> Failed : "error"
Paused --> Cancelled : "cancel signal"
Cancelled --> [*]
Completed --> [*]
Failed --> [*]
```

**Diagram sources**
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L38-L621)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L62-L1485)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L92-L244)

**Section sources**
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L38-L621)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L62-L1485)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L92-L244)

### Deterministic Execution and Checkpoints
- Version gating ensures replay determinism for memory, compression, P2P, and supervisor enhancements.
- Checkpoints are placed around decomposition, subtask execution, synthesis, and completion.
- ControlHandler.CheckPausePoint yields to process signals and blocks when paused.

```mermaid
flowchart TD
Start(["Workflow Start"]) --> V["Version checks"]
V --> Decomp["Decompose task"]
Decomp --> CP1["Checkpoint: pre_subtask_i"]
CP1 --> Exec["Execute subtask"]
Exec --> Persist["Persist agent/tool execution"]
Persist --> CP2["Checkpoint: pre_synthesis"]
CP2 --> Synth["Synthesize results"]
Synth --> CP3["Checkpoint: pre_completion"]
CP3 --> Done(["Workflow Completed"])
```

**Diagram sources**
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L420-L1027)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L281-L582)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L187-L244)

**Section sources**
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L420-L1027)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L281-L582)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L187-L244)

### Retry Mechanisms and Failure Recovery
- SimpleTaskWorkflow:
  - Fewer retries for simple tasks; consolidation activity retries once.
- SupervisorWorkflow:
  - Per-subtask retry with a maximum per-task limit.
  - Global failure threshold: allow up to half the subtasks plus one to fail before aborting.
  - Emits progress/budget events during retries.
- Intelligent retry strategy prevents infinite loops while supporting complex tasks.

```mermaid
flowchart TD
A["Start subtask"] --> E["Execute activity"]
E --> OK{"Success?"}
OK -- Yes --> Next["Proceed to next subtask"]
OK -- No --> Inc["Increment retry counter"]
Inc --> Limit{"Reached max per-task retries?"}
Limit -- Yes --> Fail["Mark subtask failed"]
Fail --> Threshold{"Too many failures?"}
Threshold -- Yes --> Abort["Abort workflow"]
Threshold -- No --> Next
Limit -- No --> E
```

**Diagram sources**
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L962-L993)

**Section sources**
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L71-L77)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L962-L993)

### Control Signals: Pause/Resume/Cancel
- Signal names and payloads are defined for pause, resume, and cancel.
- SignalHandler registers channels and a query handler for control state.
- Propagation to child workflows ensures coordinated control across nested workflows.

```mermaid
sequenceDiagram
participant Admin as "Operator/Admin"
participant Parent as "Parent Workflow"
participant Child as "Child Workflow"
participant Handler as "SignalHandler"
Admin->>Parent : Send pause_v1
Parent->>Handler : handlePause()
Handler->>Child : SignalExternalWorkflow(pause_v1)
Note over Parent,Child : Workflow blocks at next checkpoint
Admin->>Parent : Send resume_v1
Parent->>Handler : handleResume()
Handler->>Child : SignalExternalWorkflow(resume_v1)
Note over Parent,Child : Workflow resumes
Admin->>Parent : Send cancel_v1
Parent->>Handler : handleCancel()
Handler->>Child : SignalExternalWorkflow(cancel_v1)
Note over Parent,Child : Workflow cancels at checkpoint
```

**Diagram sources**
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go#L5-L41)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L31-L185)
- [control_signals.go](file://go/orchestrator/internal/workflows/control_signals.go#L7-L9)

**Section sources**
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go#L5-L41)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L31-L185)
- [control_signals.go](file://go/orchestrator/internal/workflows/control_signals.go#L7-L9)

### Task Metadata, Timestamps, and Audit Trails
- TaskExecution captures status, timestamps, model/provider, token usage, performance metrics, and metadata.
- AgentExecution and ToolExecution persist per-agent and per-tool details.
- AuditLog stores actions, IP/user-agent, request ID, and value diffs.
- Workflows emit structured events with timestamps and payloads for observability.

```mermaid
erDiagram
TASK_EXECUTION {
uuid id PK
string workflow_id
uuid user_id
string session_id
uuid tenant_id
string query
string mode
string status
timestamp started_at
timestamp completed_at
string result
string error_message
string model_used
string provider
int total_tokens
int prompt_tokens
int completion_tokens
float total_cost_usd
int duration_ms
int agents_used
int tools_invoked
int cache_hits
float complexity_score
jsonb metadata
timestamp created_at
string trigger_type
uuid schedule_id
}
AGENT_EXECUTION {
string id PK
string workflow_id
string task_id
string agent_id
string input
string output
string state
string error_message
int tokens_used
string model_used
int duration_ms
jsonb metadata
timestamp created_at
timestamp updated_at
}
TOOL_EXECUTION {
string id PK
string workflow_id
string agent_id
string agent_execution_id
string tool_name
jsonb input_params
string output
boolean success
string error
int duration_ms
int tokens_consumed
jsonb metadata
timestamp created_at
}
AUDIT_LOG {
uuid id PK
uuid user_id
string action
string entity_type
string entity_id
string ip_address
string user_agent
string request_id
jsonb old_value
jsonb new_value
timestamp created_at
}
TASK_EXECUTION ||--o{ AGENT_EXECUTION : "has"
TASK_EXECUTION ||--o{ TOOL_EXECUTION : "has"
```

**Diagram sources**
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L219)

**Section sources**
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L219)

### Practical Examples

- Monitoring task progress:
  - Subscribe to SSE/WS endpoints exposed by the admin HTTP server to receive structured events (progress, dependency satisfied, budget usage).
  - Use the control state query handler to poll current pause/cancel state.

- Handling timeouts:
  - SupervisorWorkflow implements P2P dependency waits with configurable timeouts and exponential backoff.
  - Activities specify StartToCloseTimeouts to bound execution time.

- Managing concurrent executions:
  - Workers run on multiple queues (priority or single) with configurable concurrency.
  - SupervisorWorkflow coordinates child workflows with parent-close policy to request cancellation on parent termination.

- Task cancellation and graceful shutdown:
  - SignalHandler propagates cancel to children and returns a CanceledError to mark workflow status correctly.
  - Orchestrator main process handles OS signals and gracefully stops gRPC and Temporal workers.

- Resource cleanup:
  - Persistence activities ensure agent/tool execution records are written reliably.
  - Session updates aggregate token usage and maintain budgets.

**Section sources**
- [main.go](file://go/orchestrator/main.go#L610-L799)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L642-L763)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L144-L185)

### Debugging Workflow Issues
- Use control state query handler to inspect pause/cancel state.
- Inspect emitted events around checkpoints to locate where a workflow paused or cancelled.
- Review TaskExecution/AgentExecution/ToolExecution records for token usage, durations, and errors.
- Temporal logs are bridged via ZapAdapter for consistent logging.

**Section sources**
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L42-L44)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L152)
- [adapter.go](file://go/orchestrator/internal/temporal/adapter.go#L11-L90)

## Dependency Analysis
- Orchestrator main initializes Temporal client, workers, and HTTP/admin endpoints.
- Workflows depend on activities for memory retrieval, tool execution, synthesis, and persistence.
- SignalHandler depends on control signal definitions and Temporal channels.

```mermaid
graph LR
MAIN["main.go"] --> TEMP["Temporal Client"]
MAIN --> WORKER["Temporal Worker"]
WORKER --> SUP["SupervisorWorkflow"]
WORKER --> SIM["SimpleTaskWorkflow"]
SUP --> ACT["Activities"]
SIM --> ACT
ACT --> DB["DB Models"]
SUP --> CTRLH["SignalHandler"]
SIM --> CTRLH
CTRLH --> SIG["Control Signals"]
```

**Diagram sources**
- [main.go](file://go/orchestrator/main.go#L610-L799)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L40-L1635)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L19-L664)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L14-L279)
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go#L5-L41)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L152)

**Section sources**
- [main.go](file://go/orchestrator/main.go#L610-L799)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L40-L1635)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L19-L664)
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L14-L279)
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go#L5-L41)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L152)

## Performance Considerations
- Deterministic version gates minimize replay overhead.
- Compression and context shaping reduce token usage and improve throughput.
- Budget-aware agent execution and per-agent token tracking help control costs.
- Parallel signal propagation to children avoids sequential blocking.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
- If a workflow appears stuck, check control state query handler and emitted events around checkpoints.
- Verify activity timeouts and retry policies; adjust StartToCloseTimeouts as needed.
- Confirm child workflow registration for signal propagation when debugging nested workflows.
- Review TaskExecution and AgentExecution records for token usage anomalies.

**Section sources**
- [handler.go](file://go/orchestrator/internal/workflows/control/handler.go#L42-L44)
- [simple_workflow.go](file://go/orchestrator/internal/workflows/simple_workflow.go#L71-L77)
- [supervisor_workflow.go](file://go/orchestrator/internal/workflows/supervisor_workflow.go#L962-L993)
- [models.go](file://go/orchestrator/internal/db/models.go#L61-L152)

## Conclusion
The task lifecycle integrates deterministic workflows, structured checkpoints, robust retry and failure handling, and comprehensive control signals. Persistence models and event streams provide rich observability and auditability, enabling effective monitoring, debugging, and operational control.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Appendix A: Control Signals Reference
- pause_v1: Payload includes reason and requested_by.
- resume_v1: Payload includes reason and requested_by.
- cancel_v1: Payload includes reason and requested_by.
- control_state_v1: Query handler returns current pause/cancel state.

**Section sources**
- [signals.go](file://go/orchestrator/internal/workflows/control/signals.go#L5-L41)