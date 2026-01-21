# Template Execution Engine

<cite>
**Referenced Files in This Document**
- [types.go](file://go/orchestrator/internal/templates/types.go)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go)
- [validation.go](file://go/orchestrator/internal/templates/validation.go)
- [loader.go](file://go/orchestrator/internal/templates/loader.go)
- [registry.go](file://go/orchestrator/internal/templates/registry.go)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go)
- [sequential.go](file://go/orchestrator/internal/workflows/patterns/execution/sequential.go)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go)
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

## Introduction
This document describes the template execution engine that powers workflow automation in Shannon. It explains the template runtime architecture, node execution strategies, and workflow orchestration patterns. It documents the template registry system, template loading mechanisms, and execution context management. It covers different node types (simple, cognitive, DAG, supervisor) and their execution behaviors, details the template compilation pipeline, executable plan generation, and runtime parameter resolution, and provides examples of template execution flow, error handling, performance optimization, template session management, state persistence, and execution monitoring.

## Project Structure
The template execution engine spans two primary areas:
- Template definition, validation, compilation, and registry
- Workflow orchestration that executes compiled templates with deterministic node strategies

```mermaid
graph TB
subgraph "Templates"
TTypes["types.go<br/>Template, Node, Edge, Defaults, Failure"]
TLoader["loader.go<br/>LoadTemplateFromFile, LoadTemplate"]
TValidation["validation.go<br/>ValidateTemplate"]
TCompiler["compiler.go<br/>CompileTemplate, ExecutablePlan"]
TRegistry["registry.go<br/>Registry, LoadDirectory, Finalize"]
end
subgraph "Workflows"
TWf["template_workflow.go<br/>TemplateWorkflow, executeTemplateNode,<br/>node executors"]
Seq["sequential.go<br/>ExecuteSequential"]
Par["parallel.go<br/>ExecuteParallel"]
Hyb["hybrid.go<br/>ExecuteHybrid"]
end
TTypes --> TValidation
TValidation --> TCompiler
TLoader --> TValidation
TRegistry --> TValidation
TRegistry --> TCompiler
TCompiler --> TWf
TWf --> Seq
TWf --> Par
TWf --> Hyb
```

**Diagram sources**
- [types.go](file://go/orchestrator/internal/templates/types.go#L24-L76)
- [loader.go](file://go/orchestrator/internal/templates/loader.go#L11-L42)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)
- [sequential.go](file://go/orchestrator/internal/workflows/patterns/execution/sequential.go#L47-L395)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L48-L450)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L45-L161)

**Section sources**
- [types.go](file://go/orchestrator/internal/templates/types.go#L1-L77)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L1-L172)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L1-L256)
- [loader.go](file://go/orchestrator/internal/templates/loader.go#L1-L43)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L1-L478)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L1-L851)
- [sequential.go](file://go/orchestrator/internal/workflows/patterns/execution/sequential.go#L1-L475)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L1-L520)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L1-L408)

## Core Components
- Template data model: nodes, edges, defaults, and failure policies
- Validation: structural and semantic checks, cycle detection
- Compilation: transform validated templates into deterministic executable plans
- Registry: load, deduplicate, merge inheritance, and serve templates
- Workflow orchestration: compile plan, resolve runtime context, execute nodes, aggregate results, update sessions, and emit events

**Section sources**
- [types.go](file://go/orchestrator/internal/templates/types.go#L24-L76)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)

## Architecture Overview
The engine follows a layered design:
- Templates define workflows as YAML with nodes and edges
- Validation ensures correctness and acyclicity
- Compilation builds an ExecutablePlan with adjacency, in-degree, and topological order
- Registry manages template lifecycle and inheritance composition
- Workflow orchestrator executes nodes deterministically, applying strategies and managing context, budgets, and session updates

```mermaid
sequenceDiagram
participant Client as "Client"
participant Registry as "Registry"
participant Compiler as "Compiler"
participant WF as "TemplateWorkflow"
participant Node as "Node Executor"
participant Session as "Session Update"
Client->>Registry : LoadDirectory(templateDir)
Registry-->>Client : Loaded entries
Client->>Compiler : CompileTemplate(template)
Compiler-->>Client : ExecutablePlan
Client->>WF : Start TemplateWorkflow(input)
WF->>Registry : Get(templateKey)
Registry-->>WF : Template entry
WF->>Compiler : CompileTemplate(entry.Template)
Compiler-->>WF : ExecutablePlan
loop Topological Order
WF->>Node : executeTemplateNode(node)
Node-->>WF : TemplateNodeResult
end
WF->>Session : updateTemplateSession(...)
Session-->>WF : OK
WF-->>Client : TaskResult
```

**Diagram sources**
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)
- [sequential.go](file://go/orchestrator/internal/workflows/patterns/execution/sequential.go#L47-L395)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L48-L450)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L45-L161)

## Detailed Component Analysis

### Template Data Model and Types
- Node types: simple, cognitive, DAG, supervisor
- Strategy types: react, chain_of_thought, tree_of_thoughts, debate, reflection
- Template structure includes defaults, nodes, edges, metadata
- Node-level failure policy supports degrade-to strategy, retry count, escalate-to node type

```mermaid
classDiagram
class Template {
+string Name
+string Description
+string Version
+string[] Extends
+TemplateDefaults Defaults
+TemplateNode[] Nodes
+TemplateEdge[] Edges
+map~string,any~ Metadata
+NodeByID(id) TemplateNode*
}
class TemplateDefaults {
+string ModelTier
+int BudgetAgentMax
+bool* RequireApproval
}
class TemplateNode {
+string ID
+NodeType Type
+StrategyType Strategy
+string[] DependsOn
+int* BudgetMax
+string[] ToolsAllowlist
+TemplateNodeFailure* OnFail
+map~string,any~ Metadata
}
class TemplateEdge {
+string From
+string To
}
class TemplateNodeFailure {
+StrategyType DegradeTo
+int Retry
+NodeType EscalateTo
}
class ExecutableNode {
+string ID
+NodeType Type
+StrategyType Strategy
+int BudgetMax
+string[] ToolsAllowlist
+TemplateNodeFailure* OnFail
+map~string,any~ Metadata
+string[] DependsOn
}
class ExecutablePlan {
+string TemplateName
+string TemplateVersion
+TemplateDefaults Defaults
+map~string,ExecutableNode~ Nodes
+string[] Order
+map~string,string[]~ Adjacency
+string Checksum
}
Template --> TemplateNode
Template --> TemplateEdge
Template --> TemplateDefaults
Template --> TemplateNodeFailure
ExecutablePlan --> ExecutableNode
```

**Diagram sources**
- [types.go](file://go/orchestrator/internal/templates/types.go#L24-L76)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L8-L29)

**Section sources**
- [types.go](file://go/orchestrator/internal/templates/types.go#L3-L11)
- [types.go](file://go/orchestrator/internal/templates/types.go#L13-L22)
- [types.go](file://go/orchestrator/internal/templates/types.go#L24-L76)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L8-L29)

### Template Loading and Registry
- LoadDirectory walks a directory, filters YAML files, and loads each template
- LoadTemplateFromFile and LoadTemplate decode YAML into Template structs
- Registry stores entries keyed by normalized name@version, tracks content hash, and supports listing and lookup
- Finalize resolves inheritance chains and merges overlays into base templates, validating after composition

```mermaid
flowchart TD
Start(["LoadDirectory(root)"]) --> Walk["WalkDir(root)"]
Walk --> Filter{"Is YAML?"}
Filter --> |No| Next["Skip"]
Filter --> |Yes| Read["ReadFile(path)"]
Read --> Decode["LoadTemplate(reader)"]
Decode --> Validate{"Extends empty?"}
Validate --> |Yes| V["ValidateTemplate(tpl)"]
Validate --> |No| SkipV["Skip validation here"]
V --> Key["MakeKey(name, version)"]
SkipV --> Key
Key --> Store["Registry.templates[key] = Entry{Template, Hash, SourcePath, LoadedAt}"]
Store --> End(["Done"])
```

**Diagram sources**
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [loader.go](file://go/orchestrator/internal/templates/loader.go#L11-L42)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)

**Section sources**
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L219-L242)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L244-L288)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L290-L317)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L319-L359)
- [loader.go](file://go/orchestrator/internal/templates/loader.go#L11-L42)

### Template Validation
- Validates presence of name, nodes, defaults, and budgets
- Enforces allowed node and strategy types
- Detects cycles in the graph using DFS-based cycle detection
- Ensures dependency integrity and edge validity

```mermaid
flowchart TD
VStart(["ValidateTemplate(tpl)"]) --> CheckName["Check name and nodes"]
CheckName --> CheckDefaults["Check defaults and budgets"]
CheckNodes["Iterate nodes: type, strategy, budget, depends_on, on_fail"]
CheckNodes --> BuildAdj["Build adjacency from depends_on and edges"]
BuildAdj --> Cycle["DFS findCycle()"]
Cycle --> Issues{"Any issues?"}
Issues --> |Yes| Sort["Sort issues by code"]
Sort --> ReturnErr["Return ValidationError"]
Issues --> |No| VEnd(["Valid"])
```

**Diagram sources**
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L198-L255)

**Section sources**
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L198-L255)

### Template Compilation Pipeline
- Compiles a validated Template into an ExecutablePlan
- Copies node-level attributes with cloning to avoid mutation
- Builds adjacency lists and computes in-degree
- Computes topological order; detects cycles
- Produces a stable execution order and adjacency map

```mermaid
flowchart TD
CStart(["CompileTemplate(tpl)"]) --> Validate["ValidateTemplate(tpl)"]
Validate --> Init["Init ExecutablePlan"]
Init --> CopyNodes["Copy nodes with defaults, budget, tools, on_fail, metadata"]
CopyNodes --> BuildEdges["Add edges from depends_on and explicit edges"]
BuildEdges --> Adj["Build Adjacency and in-degree"]
Adj --> Topo["topologicalOrder(adjacency, in-degree)"]
Topo --> Success{"No cycle?"}
Success --> |Yes| Done["Return ExecutablePlan"]
Success --> |No| Err["Return error: cycle detected"]
```

**Diagram sources**
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L124-L152)

**Section sources**
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L124-L152)

### Workflow Orchestration and Node Execution Strategies
- TemplateWorkflow loads the template from the registry, compiles it, and executes nodes in topological order
- Runtime context is merged from task context and node metadata, with node outputs propagated
- Node execution strategies:
  - Simple: executes a single agent task with suggested tools
  - Cognitive: applies pattern execution with budget-aware degradation
  - DAG: hybrid execution supporting parallelism, dependencies, and aggregation
  - Supervisor: spawns a child workflow with isolated context and parameters
- Aggregates agent metadata, computes cost estimates, and updates session state

```mermaid
sequenceDiagram
participant WF as "TemplateWorkflow"
participant RT as "templateRuntime"
participant Node as "executeTemplateNode"
participant Simple as "executeSimpleTemplateNode"
participant Cog as "executeCognitiveTemplateNode"
participant DAG as "executeDAGTemplateNode"
participant Sup as "executeSupervisorTemplateNode"
WF->>RT : initialize runtime
loop for nodeID in plan.Order
WF->>Node : executeTemplateNode(node)
alt NodeType == simple
Node->>Simple : ExecuteSimpleTask
Simple-->>Node : TemplateNodeResult
else NodeType == cognitive
Node->>Cog : Pattern.Execute(...)
Cog-->>Node : TemplateNodeResult
else NodeType == dag
Node->>DAG : ExecuteHybrid(...)
DAG-->>Node : TemplateNodeResult
else NodeType == supervisor
Node->>Sup : ChildWorkflow(SupervisorWorkflow)
Sup-->>Node : TemplateNodeResult
end
Node-->>RT : RecordNodeResult
end
RT-->>WF : FinalResult + metadata
WF-->>WF : updateTemplateSession + emit events
```

**Diagram sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L229-L250)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L252-L292)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L294-L368)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L370-L445)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L447-L506)

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L229-L250)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L252-L292)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L294-L368)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L370-L445)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L447-L506)

### Simple Node Execution
- Merges context and passes query/history/session context
- Executes a single agent task with suggested tools
- Records agent results and returns node result with tokens and metadata

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L252-L292)

### Cognitive Node Execution
- Determines effective strategy considering budget thresholds and degrades when needed
- Resolves model tier from defaults or node metadata
- Executes registered pattern and aggregates agent results

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L294-L368)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L508-L521)

### DAG Node Execution (Hybrid)
- Parses tasks from node metadata or falls back to dependency aggregation
- Supports parallel execution with configurable concurrency and budget per agent
- Supports dependency wait with incremental timeouts and optional passing of dependency results
- Summarizes results and metadata for downstream consumption

```mermaid
flowchart TD
HStart(["parseHybridTasks(metadata)"]) --> HasTasks{"tasks found?"}
HasTasks --> |No| Aggregate["aggregateDependencyOutputs(rt, node)"]
HasTasks --> |Yes| Config["build HybridConfig (concurrency, events, pass results)"]
Config --> Budget["compute budget_per_agent"]
Budget --> Execute["ExecuteHybrid(...)"]
Execute --> Summarize["summariseHybridResult(...)"]
Aggregate --> Return["return TemplateNodeResult"]
Summarize --> Return
```

**Diagram sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L606-L640)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L370-L445)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L754-L786)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L45-L161)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L48-L450)

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L606-L640)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L370-L445)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L754-L786)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L45-L161)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L48-L450)

### Supervisor Node Execution
- Spawns a child workflow with isolated context and parameters
- Propagates tools, tool parameters, mode, and approval settings from node metadata
- Returns child result with merged metadata

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L447-L506)

### Execution Context Management and Parameter Resolution
- Runtime context merging: task context + node metadata
- Node outputs passed to subsequent nodes for dependency-driven workflows
- Query determination: metadata.query overrides default query
- Model tier resolution: node metadata overrides defaults
- Numeric metadata helpers: booleans, integers, string slices, and maps are parsed from metadata

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L229-L250)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L230-L234)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L523-L532)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L508-L521)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L586-L730)

### Template Session Management, State Persistence, and Monitoring
- Session updates: after workflow completion, updates result, tokens, and agent usage
- Vector store recording: records query-answer pairs with template metadata
- Streaming events: emits workflow started/completed and agent events during execution
- Token usage recording: standardized activity for billing and observability

```mermaid
sequenceDiagram
participant WF as "TemplateWorkflow"
participant Act as "Activities"
participant Mon as "Metrics/Events"
WF->>Act : UpdateSessionResult(sessionID, result, tokens, agents, usages)
Act-->>WF : OK
WF->>Mon : EmitTaskUpdate(StreamEventWorkflowCompleted)
WF->>Act : RecordQuery(query, answer, model, metadata)
Act-->>WF : OK
```

**Diagram sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L146-L171)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L788-L850)

**Section sources**
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L146-L171)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L788-L850)

## Dependency Analysis
- Templates depend on validation and compilation
- Registry depends on loader and validation
- Workflow orchestration depends on templates and execution patterns
- Execution patterns depend on activities and agent execution

```mermaid
graph LR
Loader["loader.go"] --> Validation["validation.go"]
Validation --> Compiler["compiler.go"]
Loader --> Registry["registry.go"]
Validation --> Registry
Registry --> Compiler
Compiler --> TemplateWF["template_workflow.go"]
TemplateWF --> Sequential["sequential.go"]
TemplateWF --> Parallel["parallel.go"]
TemplateWF --> Hybrid["hybrid.go"]
```

**Diagram sources**
- [loader.go](file://go/orchestrator/internal/templates/loader.go#L11-L42)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)
- [sequential.go](file://go/orchestrator/internal/workflows/patterns/execution/sequential.go#L47-L395)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L48-L450)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L45-L161)

**Section sources**
- [loader.go](file://go/orchestrator/internal/templates/loader.go#L11-L42)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L68-L196)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L31-L122)
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L51-L162)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L31-L172)
- [sequential.go](file://go/orchestrator/internal/workflows/patterns/execution/sequential.go#L47-L395)
- [parallel.go](file://go/orchestrator/internal/workflows/patterns/execution/parallel.go#L48-L450)
- [hybrid.go](file://go/orchestrator/internal/workflows/patterns/execution/hybrid.go#L45-L161)

## Performance Considerations
- Deterministic topological ordering ensures predictable execution and replayability
- Budget-aware strategy degradation prevents expensive cognitive patterns from exceeding limits
- Concurrency control via semaphores and child workflow options balances throughput and resource usage
- Incremental dependency waiting avoids long single timeouts and improves UI responsiveness
- Token usage recording and cost estimation enable observability and cost control

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and diagnostics:
- Template load failures: registry aggregates load errors; inspect failures list and content hash mismatches
- Validation errors: ValidationError with structured issues; check for missing names, invalid node types, cycles, and budget constraints
- Compilation errors: cycle detection in topological sort; fix edges/depends_on
- Node execution failures: per-node result includes success/error; review node type and strategy configuration
- Session update failures: activity errors during session persistence; verify session ID and agent usage inputs

**Section sources**
- [registry.go](file://go/orchestrator/internal/templates/registry.go#L460-L477)
- [validation.go](file://go/orchestrator/internal/templates/validation.go#L15-L50)
- [compiler.go](file://go/orchestrator/internal/templates/compiler.go#L148-L151)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L115-L124)
- [template_workflow.go](file://go/orchestrator/internal/workflows/template_workflow.go#L807-L820)

## Conclusion
The template execution engine provides a robust, deterministic framework for defining and executing automated workflows. It combines YAML-based template authoring with strong validation, inheritance composition, and executable plan compilation. The workflow orchestrator applies node-specific strategies, manages context and budgets, and integrates tightly with session management, monitoring, and cost tracking. This design enables scalable, observable, and maintainable automation across diverse domains.