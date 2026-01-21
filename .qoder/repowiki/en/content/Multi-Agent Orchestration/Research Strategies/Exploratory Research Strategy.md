# Exploratory Research Strategy

<cite>
**Referenced Files in This Document**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go)
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go)
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go)
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go)
- [config.go](file://go/orchestrator/internal/activities/config.go)
- [research_strategies.yaml](file://config/research_strategies.yaml)
- [features.yaml](file://config/features.yaml)
- [market_analysis.yaml](file://config/workflows/examples/market_analysis.yaml)
- [types.go](file://go/orchestrator/internal/workflows/strategies/types.go)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go)
- [task.go](file://go/orchestrator/cmd/gateway/internal/handlers/task.go)
- [registry.go](file://go/orchestrator/cmd/gateway/internal/openai/registry.go)
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
This document explains the Exploratory Research Strategy implementation that enables discovery through open-ended exploration, hypothesis generation, and unexpected insight identification. It documents how the system minimizes research constraints, maximizes serendipitous discoveries, and maintains analytical flexibility. The strategy leverages adaptive search patterns, emerging trend identification, and cross-domain connection finding. It also covers practical workflows for innovation research, market opportunity discovery, and academic exploration phases, along with provider selection, cost management, and integration with subsequent structured research phases.

## Project Structure
The Exploratory Research Strategy is implemented as a Temporal workflow orchestrated by the gateway and executed by cognitive patterns. Configuration is centralized in YAML and Viper-based loaders. The workflow integrates memory retrieval, context compression, and optional citation formatting, then applies debate and reflection to refine results.

```mermaid
graph TB
GW["Gateway Handlers<br/>session.go, task.go"] --> EXW["ExploratoryWorkflow<br/>exploratory.go"]
EXW --> CFG["Workflow Config Loader<br/>config.go"]
EXW --> PAT_TOT["Tree-of-Thoughts Pattern<br/>tree_of_thoughts.go"]
EXW --> PAT_DEBATE["Debate Pattern<br/>debate.go"]
EXW --> PAT_REFLECT["Reflection Pattern<br/>reflection.go"]
EXW --> MEM["Memory Retrieval<br/>Hierarchical/Session"]
EXW --> COMP["Context Compression"]
EXW --> PRICING["Pricing/Cost Estimation"]
EXW --> TYPES["Workflow Types<br/>types.go"]
EXW --> STRAT_YAML["Research Strategies YAML<br/>research_strategies.yaml"]
EXW --> FEAT_YAML["Features YAML<br/>features.yaml"]
```

**Diagram sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L1-L426)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go#L1-L631)
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go#L1-L644)
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L1-L170)
- [research_strategies.yaml](file://config/research_strategies.yaml#L1-L53)
- [features.yaml](file://config/features.yaml#L65-L105)
- [types.go](file://go/orchestrator/internal/workflows/strategies/types.go#L1-L54)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L56-L295)
- [task.go](file://go/orchestrator/cmd/gateway/internal/handlers/task.go#L35-L351)

**Section sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L1-L426)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)
- [research_strategies.yaml](file://config/research_strategies.yaml#L1-L53)
- [features.yaml](file://config/features.yaml#L65-L105)

## Core Components
- ExploratoryWorkflow orchestrates three-phase exploration: Tree-of-Thoughts for systematic branching, optional Debate for multi-perspective validation, and Reflection for quality refinement. It injects memory, compresses context when needed, and emits streaming updates.
- Tree-of-Thoughts generates reasoning branches with configurable depth and branching factor, prunes low-scoring paths, and backtracks when confidence is low.
- Debate pattern runs multiple agents representing different perspectives, iterates rounds, and synthesizes a final position via moderator or voting.
- Reflection pattern evaluates results against criteria and resynthesizes with feedback until a confidence threshold is met.
- Configuration is loaded from features.yaml and research_strategies.yaml, controlling iteration counts, branching factors, confidence thresholds, and model tiers.

**Section sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L17-L426)
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go#L52-L236)
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go#L48-L473)
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L17-L169)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)
- [research_strategies.yaml](file://config/research_strategies.yaml#L12-L53)
- [features.yaml](file://config/features.yaml#L65-L105)

## Architecture Overview
The Exploratory Research Strategy integrates gateway input, workflow orchestration, cognitive patterns, memory systems, and cost tracking. It supports streaming updates and control signals for pause/resume/cancel.

```mermaid
sequenceDiagram
participant Client as "Client"
participant Gateway as "Gateway Handlers"
participant Workflow as "ExploratoryWorkflow"
participant Patterns as "Patterns (ToT/Debate/Reflect)"
participant Memory as "Memory Systems"
participant Pricing as "Pricing Engine"
Client->>Gateway : Submit task with research_strategy
Gateway->>Workflow : Dispatch with context and budget
Workflow->>Memory : Fetch hierarchical/session memory
Workflow->>Patterns : Tree-of-Thoughts exploration
Patterns-->>Workflow : Best path and confidence
alt Confidence low
Workflow->>Patterns : Debate with perspectives
Patterns-->>Workflow : Enhanced position
end
alt Final confidence low
Workflow->>Patterns : Reflection with feedback
Patterns-->>Workflow : Refined result
end
Workflow->>Pricing : Compute cost estimate
Workflow-->>Gateway : Stream updates and final result
Gateway-->>Client : Response with metadata and tokens
```

**Diagram sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L56-L295)
- [task.go](file://go/orchestrator/cmd/gateway/internal/handlers/task.go#L35-L351)
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L19-L426)
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go#L52-L236)
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go#L48-L473)
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L17-L169)

## Detailed Component Analysis

### ExploratoryWorkflow
- Input validation and control signal setup for pause/resume/cancel.
- Loads configuration for iterations, branching, and concurrency.
- Memory injection via hierarchical or session memory retrieval.
- Context compression when message volume exceeds thresholds.
- Executes Tree-of-Thoughts with pruning and optional backtracking.
- Applies Debate when confidence is below threshold.
- Applies Reflection to reach near-perfect confidence.
- Emits streaming updates and final metadata including cost estimation.

```mermaid
flowchart TD
Start(["Start ExploratoryWorkflow"]) --> Validate["Validate Input"]
Validate --> LoadCfg["Load Workflow Config"]
LoadCfg --> SetupSignals["Setup Control Signals"]
SetupSignals --> InjectMemory["Fetch Hierarchical/Session Memory"]
InjectMemory --> CompressCheck{"Context Needs Compression?"}
CompressCheck --> |Yes| Compress["Compress Context"]
CompressCheck --> |No| PreExecCheck["Check Pause/Cancel"]
Compress --> PreExecCheck
PreExecCheck --> TreeOfThoughts["Execute Tree-of-Thoughts"]
TreeOfThoughts --> ConfidenceCheck{"Confidence < Threshold?"}
ConfidenceCheck --> |Yes| Debate["Run Debate Pattern"]
ConfidenceCheck --> |No| ReflectCheck{"Final Confidence < 0.9?"}
Debate --> ReflectCheck
ReflectCheck --> |Yes| Reflection["Run Reflection Pattern"]
ReflectCheck --> |No| FormatCitations["Format Citations if Enabled"]
Reflection --> FormatCitations
FormatCitations --> UpdateSession["Update Session"]
UpdateSession --> EmitFinal["Emit Final Output and Completion"]
EmitFinal --> End(["Complete"])
```

**Diagram sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L19-L426)

**Section sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L19-L426)

### Tree-of-Thoughts Pattern
- Generates reasoning branches from current nodes with configurable branching factor.
- Evaluates nodes using heuristics (solution indicators, logical progression, concreteness).
- Prunes low-scoring branches and backtracks to promising alternatives when confidence is low.
- Synthesizes final solution from the best path and computes confidence with depth penalties.

```mermaid
flowchart TD
Init(["Initialize Root Node"]) --> Loop{"Queue Not Empty<br/>and Budget Available?"}
Loop --> |Yes| Sort["Sort Queue by Score"]
Sort --> Pop["Pop Highest-Score Node"]
Pop --> DepthCheck{"Depth < MaxDepth?"}
DepthCheck --> |No| MarkTerminal["Mark Terminal"] --> Loop
DepthCheck --> |Yes| BranchGen["Generate Branches"]
BranchGen --> Eval["Evaluate and Score"]
Eval --> Prune{"Score >= Threshold?"}
Prune --> |No| Loop
Prune --> |Yes| Add["Add to Tree and Queue"]
Add --> Loop
Loop --> |No| BestPath["Find Best Path"]
BestPath --> Synthesize["Synthesize Solution"]
Synthesize --> Confidence["Calculate Confidence"]
Confidence --> Backtrack{"Backtrack Enabled<br/>and Low Confidence?"}
Backtrack --> |Yes| ExploreAlt["Explore Top Alternatives"]
Backtrack --> |No| Done(["Return Result"])
ExploreAlt --> Done
```

**Diagram sources**
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go#L52-L236)

**Section sources**
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go#L52-L236)

### Debate Pattern
- Runs multiple debaters with distinct perspectives, collects initial positions, and iterates rounds of counter-arguments and strengthening.
- Supports moderator synthesis or voting to resolve positions.
- Records token usage and persists debate consensus for learning.

```mermaid
sequenceDiagram
participant EXW as "ExploratoryWorkflow"
participant DEBATE as "Debate Pattern"
participant D1 as "Debater 1"
participant D2 as "Debater 2"
participant D3 as "Debater 3"
EXW->>DEBATE : Initialize with Perspectives
DEBATE->>D1 : Initial Position Prompt
DEBATE->>D2 : Initial Position Prompt
DEBATE->>D3 : Initial Position Prompt
D1-->>DEBATE : Position + Arguments
D2-->>DEBATE : Position + Arguments
D3-->>DEBATE : Position + Arguments
DEBATE->>D1 : Round Response Prompt
DEBATE->>D2 : Round Response Prompt
DEBATE->>D3 : Round Response Prompt
D1-->>DEBATE : Updated Position
D2-->>DEBATE : Updated Position
D3-->>DEBATE : Updated Position
DEBATE-->>EXW : Final Position (Moderator/Vote/Synthesis)
```

**Diagram sources**
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go#L48-L473)

**Section sources**
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go#L48-L473)

### Reflection Pattern
- Evaluates the current result against predefined criteria and computes a quality score.
- If below threshold, constructs reflection context with feedback and resynthesizes the result.
- Tracks tokens and records usage for billing alignment.

```mermaid
flowchart TD
Start(["Start Reflection"]) --> Evaluate["Evaluate Result"]
Evaluate --> ScoreCheck{"Score >= Threshold?"}
ScoreCheck --> |Yes| Return(["Return Result"])
ScoreCheck --> |No| Retry{"Retry Count < Max Retries?"}
Retry --> |No| Return
Retry --> |Yes| Feedback["Build Reflection Context"]
Feedback --> Resynthesize["Resynthesize with Feedback"]
Resynthesize --> Evaluate
```

**Diagram sources**
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L17-L169)

**Section sources**
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L17-L169)

### Configuration and Strategy Selection
- Research strategies (quick, standard, deep, academic) define iteration limits, branching, confidence thresholds, and model tiers.
- Cognitive workflow defaults (iterations, branching, thresholds) are loaded from features.yaml and exposed via config loader.
- Gateway handlers map research_strategy into task context and enforce allowed values.

```mermaid
graph LR
RS["research_strategies.yaml"] --> GW_CTX["Gateway Context Mapping<br/>task.go, session.go"]
GW_CTX --> EXW["ExploratoryWorkflow"]
FEAT["features.yaml"] --> CFG["GetWorkflowConfig<br/>config.go"]
CFG --> EXW
EXW --> PAT_TOT["Tree-of-Thoughts"]
EXW --> PAT_DEBATE["Debate"]
EXW --> PAT_REFLECT["Reflection"]
```

**Diagram sources**
- [research_strategies.yaml](file://config/research_strategies.yaml#L12-L53)
- [features.yaml](file://config/features.yaml#L65-L105)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)
- [task.go](file://go/orchestrator/cmd/gateway/internal/handlers/task.go#L35-L351)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L56-L295)

**Section sources**
- [research_strategies.yaml](file://config/research_strategies.yaml#L12-L53)
- [features.yaml](file://config/features.yaml#L65-L105)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)
- [task.go](file://go/orchestrator/cmd/gateway/internal/handlers/task.go#L35-L351)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L56-L295)

## Dependency Analysis
The ExploratoryWorkflow depends on:
- Configuration loader for cognitive defaults and strategy presets.
- Memory retrieval for contextual grounding.
- Pricing for cost estimation aligned with token usage.
- Control signals for lifecycle management.
- Patterns for adaptive exploration, debate, and reflection.

```mermaid
graph TB
EXW["ExploratoryWorkflow"] --> CFG["GetWorkflowConfig"]
EXW --> MEM["Memory Fetch"]
EXW --> PAT_TOT["Tree-of-Thoughts"]
EXW --> PAT_DEBATE["Debate"]
EXW --> PAT_REFLECT["Reflection"]
EXW --> PRICING["Pricing/CostForTokens"]
EXW --> CTRL["Control Signals"]
EXW --> TYPES["Task Types"]
```

**Diagram sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L19-L426)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)
- [tree_of_thoughts.go](file://go/orchestrator/internal/workflows/patterns/tree_of_thoughts.go#L52-L236)
- [debate.go](file://go/orchestrator/internal/workflows/patterns/debate.go#L48-L473)
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L17-L169)

**Section sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L19-L426)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)

## Performance Considerations
- Iterative refinement reduces reliance on single large-model calls, lowering cost while maintaining quality.
- Budget-aware agent execution caps tokens per thought/round to prevent runaway consumption.
- Context compression proactively manages token budgets for long histories.
- Confidence thresholds gate additional expensive steps (Debate, Reflection) to optimize throughput.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and mitigations:
- Low confidence after Tree-of-Thoughts: The workflow triggers Debate to diversify perspectives; verify debate configuration and memory injection.
- Reflection not improving result: Check evaluation criteria and feedback; ensure sufficient retries and budget allocation.
- Streaming interruptions: Verify control signals and parent workflow ID propagation for child workflows.
- Cost overruns: Confirm model tier selection and token usage recording; adjust branching factor and max iterations.

**Section sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L209-L325)
- [reflection.go](file://go/orchestrator/internal/workflows/patterns/reflection.go#L17-L169)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)

## Conclusion
The Exploratory Research Strategy balances openness and rigor by combining systematic branching, multi-perspective debate, and iterative reflection. It minimizes constraints through adaptive search patterns, identifies emerging trends via memory and context compression, and maintains flexibility through configurable thresholds and model tiers. Integration with structured research phases ensures seamless handoff to deeper analysis while preserving cost efficiency and analytical fidelity.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Practical Workflows and Examples
- Innovation research: Use higher branching and iterations to explore novel hypotheses; apply Debate to challenge assumptions; use Reflection to strengthen conclusions.
- Market opportunity discovery: Combine Tree-of-Thoughts with parallel DAG workflows for competitive and trend analysis; integrate supervisor synthesis and reflection for final reports.
- Academic exploration: Employ academic strategy preset with generous gaps and iterations; leverage hierarchical memory and citation formatting for scholarly output.

**Section sources**
- [research_strategies.yaml](file://config/research_strategies.yaml#L41-L53)
- [market_analysis.yaml](file://config/workflows/examples/market_analysis.yaml#L1-L76)

### Provider Selection and Cost Management
- Provider selection is managed centrally; model tier presets guide agent execution while synthesis uses larger models as needed.
- Cost management is achieved through:
  - Small model tiers for utility and agent execution.
  - Budgeted agent execution to cap tokens per step.
  - Confidence-gated refinement to avoid unnecessary computation.
  - Centralized pricing for cost estimation.

**Section sources**
- [research_strategies.yaml](file://config/research_strategies.yaml#L3-L11)
- [config.go](file://go/orchestrator/internal/activities/config.go#L75-L348)

### Integration with Structured Research Phases
- The Exploratory phase produces a best path and confidence; downstream phases can:
  - Use synthesis templates for formal reporting.
  - Apply structured research patterns with stricter sourcing and verification.
  - Persist findings and citations for auditability and reuse.

**Section sources**
- [exploratory.go](file://go/orchestrator/internal/workflows/strategies/exploratory.go#L327-L333)
- [market_analysis.yaml](file://config/workflows/examples/market_analysis.yaml#L42-L61)