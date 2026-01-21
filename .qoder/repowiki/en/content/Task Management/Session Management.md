# Session Management

<cite>
**Referenced Files in This Document**
- [types.go](file://go/orchestrator/internal/session/types.go)
- [manager.go](file://go/orchestrator/internal/session/manager.go)
- [session_service.go](file://go/orchestrator/internal/server/session_service.go)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go)
- [session.go](file://go/orchestrator/internal/activities/session.go)
- [session_title.go](file://go/orchestrator/internal/activities/session_title.go)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go)
- [007_session_soft_delete.sql](file://migrations/postgres/007_session_soft_delete.sql)
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py)
- [session.proto](file://protos/session/session.proto)
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
This document explains session management across the system, focusing on session continuity and persistence. It covers the lifecycle from creation to termination, including session IDs, user and tenant associations, and tenant isolation. It documents memory storage and retrieval, context preservation across tasks, metadata and title generation, token usage tracking and budget management, multi-turn conversations, context switching, session recovery after failures, expiration and cleanup, and practical debugging and optimization guidance.

## Project Structure
Session management spans Go services, HTTP handlers, gRPC services, Redis-backed persistence, and database schema with soft-delete and dual-ID support. The Python LLM service includes session-aware tools for file operations.

```mermaid
graph TB
subgraph "Go Runtime"
SM["Session Manager<br/>Redis-backed"]
SS["gRPC Session Service"]
AH["HTTP Session Handler"]
OA["OpenAI Session Manager"]
ACT["Activities<br/>UpdateSessionResult, Title Generation"]
end
subgraph "Persistence"
R["Redis"]
PG["PostgreSQL<br/>sessions table"]
end
subgraph "Python LLM Service"
SF["Session-aware Tools<br/>session_file.py"]
end
SM --- R
SS --- SM
AH --- PG
OA --- R
ACT --- SM
ACT --- PG
SF --- SM
```

**Diagram sources**
- [manager.go](file://go/orchestrator/internal/session/manager.go#L20-L95)
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L19-L82)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L25-L111)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L38-L52)
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L111)

**Section sources**
- [manager.go](file://go/orchestrator/internal/session/manager.go#L20-L95)
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L19-L82)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L25-L111)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L38-L52)
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L111)

## Core Components
- Session model: Holds identity, user/tenant association, timestamps, metadata/context, history, agent states, and token/cost accounting.
- Session Manager: Creates, retrieves, updates, deletes, extends TTL, and cleans up sessions; uses Redis with local LRU cache and tenant isolation.
- gRPC Session Service: Exposes Create/Get/Update/Delete/AddMessage/ClearHistory RPCs backed by the Manager.
- HTTP Session Handler: Provides REST endpoints to fetch session metadata, history, and events; aggregates token usage from Redis or DB.
- Activities: Update session results (token usage, cost, history), maintain context, and generate titles.
- OpenAI Session Manager: Derives or resolves OpenAI-compatible session IDs mapped to Shannon sessions, with collision detection.
- Database Schema: Supports soft delete and dual-ID (UUID + external_id) with indexes for efficient lookups.
- Session-aware Tools: Python tools track files created during a session and expose them in context.

**Section sources**
- [types.go](file://go/orchestrator/internal/session/types.go#L19-L145)
- [manager.go](file://go/orchestrator/internal/session/manager.go#L97-L283)
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L278)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L337)
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session_title.go](file://go/orchestrator/internal/activities/session_title.go#L35-L194)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L62-L133)
- [007_session_soft_delete.sql](file://migrations/postgres/007_session_soft_delete.sql#L7-L41)
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L111)

## Architecture Overview
The session subsystem integrates HTTP/gRPC APIs, Redis-backed persistence, PostgreSQL for durable metadata and history, and activity-driven updates.

```mermaid
sequenceDiagram
participant Client as "Client"
participant Gateway as "HTTP Session Handler"
participant DB as "PostgreSQL"
participant Redis as "Redis"
Client->>Gateway : GET /api/v1/sessions/{id}
Gateway->>DB : Query session by UUID or external_id (tenant-aware)
DB-->>Gateway : Session metadata
Gateway->>Redis : Optionally read total_tokens_used
Redis-->>Gateway : Tokens (if present)
Gateway-->>Client : SessionResponse (tokens, budget, context)
```

**Diagram sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L337)

```mermaid
sequenceDiagram
participant WF as "Workflow"
participant Act as "Activities.UpdateSessionResult"
participant SM as "Session Manager"
participant Redis as "Redis"
WF->>Act : Report tokens_used, cost, result, agents_used
Act->>SM : GetSession(session_id)
SM->>Redis : Get session JSON
Redis-->>SM : Session data
SM-->>Act : Session object
Act->>Act : Compute cost (pricing), update totals
Act->>SM : Append assistant message, update context
SM->>Redis : Set session JSON with TTL
Redis-->>SM : OK
SM-->>Act : Updated session
Act-->>WF : Success
```

**Diagram sources**
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [manager.go](file://go/orchestrator/internal/session/manager.go#L186-L243)

```mermaid
sequenceDiagram
participant OA as "OpenAI Adapter"
participant OAM as "OpenAI Session Manager"
participant Redis as "Redis"
OA->>OAM : ResolveSession(provided_session_id?, user_id, tenant_id)
alt Provided session missing
OAM->>OAM : deriveSessionID(req,user_id)
end
OAM->>Redis : Get openai : session : {id}
alt Exists and owned
OAM-->>OA : {session_id, shannon_session_id}
else Collision or new
OAM->>Redis : Set openai : session : {id} (with TTL)
OAM-->>OA : {new_session_id, shannon_session_id, IsNew=true}
end
```

**Diagram sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L62-L133)

## Detailed Component Analysis

### Session Model and Lifecycle
- Identity and associations: Session carries a unique ID, associated user_id and tenant_id, creation/update/expiry timestamps, and metadata/context.
- History and agent states: Maintains a bounded message history and per-agent state snapshots.
- Token and cost tracking: Tracks total tokens used and total cost in USD; supports per-message token/cost fields.
- Expiration: Provides IsExpired() and TTL management via manager configuration and Redis TTL.

```mermaid
classDiagram
class Session {
+string ID
+string UserID
+string TenantID
+time.Time CreatedAt
+time.Time UpdatedAt
+time.Time ExpiresAt
+map[string]interface{} Metadata
+map[string]interface{} Context
+[]Message History
+map[string]*AgentState AgentStates
+int TotalTokensUsed
+float64 TotalCostUSD
+IsExpired() bool
+GetContextValue(key) (interface{}, bool)
+SetContextValue(key, value) void
+GetAgentState(agentID) (*AgentState, bool)
+SetAgentState(agentID, state) void
+GetRecentHistory(n) []Message
+GetHistorySummary(maxTokens) string
+UpdateTokenUsage(tokens, cost) void
}
class Message {
+string ID
+string Role
+string Content
+time.Time Timestamp
+map[string]interface{} Metadata
+int TokensUsed
+float64 CostUSD
}
class AgentState {
+string AgentID
+time.Time LastActive
+string State
+map[string]interface{} Memory
+[]string ToolsUsed
+int TokensUsed
}
Session --> Message : "has many"
Session --> AgentState : "has many"
```

**Diagram sources**
- [types.go](file://go/orchestrator/internal/session/types.go#L19-L145)

**Section sources**
- [types.go](file://go/orchestrator/internal/session/types.go#L19-L145)

### Session Manager: Creation, Retrieval, Updates, Cleanup
- Creation: Generates a UUID, applies TTL, initializes context/history, persists to Redis, caches locally, emits metrics.
- Retrieval: Checks local cache (LRU access tracking), otherwise loads from Redis, enforces expiration and tenant isolation, updates cache.
- Updates: Persists session JSON with computed TTL, updates local cache.
- Deletion: Removes from Redis and local cache.
- TTL extension: Loads session, updates ExpiresAt, persists.
- History management: Appends messages and enforces max history bound.
- Context updates: Sets key-value pairs in context.
- Listing: Scans Redis keyspace (pattern-based) and filters by user/tenant and expiry.
- Cleanup: Iterates keyspace, unmarshals, checks expiry, deletes expired entries.

```mermaid
flowchart TD
Start([GetSession]) --> CheckCache["Check local cache"]
CheckCache --> CacheHit{"Cache hit?"}
CacheHit --> |Yes| ReturnCache["Return cached session"]
CacheHit --> |No| LoadRedis["Load from Redis by key"]
LoadRedis --> NotFound{"Exists?"}
NotFound --> |No| ErrNotFound["ErrSessionNotFound"]
NotFound --> |Yes| CheckExpire["IsExpired()?"]
CheckExpire --> |Yes| Del["Delete session"]
Del --> ErrExpired["ErrSessionExpired"]
CheckExpire --> |No| TenantCheck["Enforce tenant isolation"]
TenantCheck --> UpdateCache["Put in cache, update access time"]
UpdateCache --> ReturnSession["Return session"]
```

**Diagram sources**
- [manager.go](file://go/orchestrator/internal/session/manager.go#L186-L243)

**Section sources**
- [manager.go](file://go/orchestrator/internal/session/manager.go#L97-L283)
- [manager.go](file://go/orchestrator/internal/session/manager.go#L328-L392)

### gRPC Session Service
- CreateSession: Accepts user_id, optional initial context, optional max_history and TTL; derives tenant from auth context; returns session_id and expires_at.
- GetSession: Retrieves session; optionally includes history; converts to protobuf.
- UpdateSession: Merges context updates; optionally extends TTL; persists and returns new expiry.
- DeleteSession: Deletes session.
- AddMessage: Appends a message to history; returns updated history size.
- ClearHistory: Clears history; optionally keeps context; persists.

```mermaid
sequenceDiagram
participant Client as "gRPC Client"
participant Svc as "SessionServiceImpl"
participant SM as "Session Manager"
Client->>Svc : CreateSession(userId, initialContext, maxHistory, ttl)
Svc->>SM : CreateSession(userID, tenantID, metadata)
SM-->>Svc : Session
Svc-->>Client : CreateSessionResponse(expiresAt)
Client->>Svc : UpdateSession(session_id, context_updates, extend_ttl)
Svc->>SM : GetSession(session_id)
SM-->>Svc : Session
Svc->>SM : ExtendSession(session_id, duration)
Svc->>SM : UpdateSession(session)
SM-->>Svc : OK
Svc-->>Client : UpdateSessionResponse(new_expires_at)
```

**Diagram sources**
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L185)
- [manager.go](file://go/orchestrator/internal/session/manager.go#L97-L184)

**Section sources**
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L278)

### HTTP Session Handler: Metadata, History, Events
- GetSession: Queries sessions table by UUID or external_id, enforces user/tenant filters and soft-delete; reads tokens_used from Redis if available, otherwise aggregates from task_executions; detects research session and strategy from context or first task metadata; returns SessionResponse.
- GetSessionHistory: Lists tasks for a session (both UUID and external_id), ordered by started_at.
- GetSessionEvents: Groups events by task (turns), computes metadata (tokens, execution time, agents), supports pagination and payload inclusion.

```mermaid
sequenceDiagram
participant Client as "Client"
participant Handler as "SessionHandler"
participant DB as "PostgreSQL"
participant Redis as "Redis"
Client->>Handler : GET /sessions/{id}
Handler->>DB : SELECT session by id or context->>external_id
DB-->>Handler : Session row
Handler->>Redis : GET session : {id} or session : {external_id}
Redis-->>Handler : total_tokens_used (optional)
Handler->>DB : SUM(total_tokens) from task_executions (fallback)
DB-->>Handler : Aggregated tokens
Handler-->>Client : SessionResponse
```

**Diagram sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L337)

**Section sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L518)

### Activities: Results, Token Usage, Context Preservation
- UpdateSessionResult: Validates input, fetches session, computes cost (per-agent/model/default), updates token totals and cost, appends assistant message, maintains context fields (last_updated_at, totals, last_*), persists session.
- Session Title Generation: Generates a concise title via LLM with fallback to query truncation; stores title in Redis via session manager and also updates Postgres sessions.context to ensure HTTP APIs can read it.

```mermaid
flowchart TD
AStart([UpdateSessionResult]) --> Validate["Validate input"]
Validate --> GetSess["GetSession(session_id)"]
GetSess --> ComputeCost["Compute cost (pricing)"]
ComputeCost --> UpdateTotals["UpdateTokenUsage(tokens_used, cost)"]
UpdateTotals --> AppendMsg["Append assistant message"]
AppendMsg --> EnforceLimit["Trim history to max"]
EnforceLimit --> UpdateContext["Update context fields"]
UpdateContext --> Persist["UpdateSession()"]
Persist --> AEnd([Done])
```

**Diagram sources**
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)

**Section sources**
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session_title.go](file://go/orchestrator/internal/activities/session_title.go#L35-L194)

### OpenAI Session Manager: Deterministic ID Resolution and Collision Handling
- ResolveSession: If no session provided, derives a deterministic ID from request content and user_id; if provided, checks ownership; on collision, generates a new session; otherwise uses existing; updates last_used and message_count.
- deriveSessionID: Hashes user_id plus selected portions of system/user messages to form a short session identifier.
- getOrCreateSession: Creates mapping from OpenAI session ID to Shannon session ID with TTL.

```mermaid
flowchart TD
RS([ResolveSession]) --> Provided{"Provided session_id?"}
Provided --> |No| Derive["deriveSessionID(req, user_id)"]
Derive --> CreateOrGet["getOrCreateSession(id, user_id, tenant_id)"]
Provided --> |Yes| Check["getSessionInfo(id)"]
Check --> Exists{"Exists and owned?"}
Exists --> |Yes| Touch["touchSession()"] --> ReturnExisting["Return {id, shannon_id}"]
Exists --> |No| NewID["generateSessionID()"] --> CreateOrGet
CreateOrGet --> ReturnNew["Return {id, shannon_id, IsNew}"]
```

**Diagram sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L62-L133)

**Section sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L62-L133)

### Database Schema: Soft Delete and Dual-ID Support
- Soft delete columns: deleted_at and deleted_by enable soft deletion with partial index.
- Dual-ID support: Functional index on context->>'external_id' and unique constraint per user ensure external_id uniqueness and efficient lookups.
- Additional indexes: Improve non-deleted session queries and task_executions session_id lookups.

```mermaid
erDiagram
SESSIONS {
uuid id PK
uuid user_id FK
text tenant_id
jsonb context
int token_budget
int tokens_used
timestamptz created_at
timestamptz updated_at
timestamptz expires_at
timestamptz deleted_at
uuid deleted_by
}
TASK_EXECUTIONS {
uuid id PK
uuid session_id FK
uuid user_id FK
text tenant_id
int total_tokens
float total_cost_usd
timestamptz started_at
timestamptz completed_at
}
SESSIONS ||--o{ TASK_EXECUTIONS : "has many"
```

**Diagram sources**
- [007_session_soft_delete.sql](file://migrations/postgres/007_session_soft_delete.sql#L7-L41)

**Section sources**
- [007_session_soft_delete.sql](file://migrations/postgres/007_session_soft_delete.sql#L1-L50)

### Session-Aware Tools: File Operations Tracking
- SessionFileWriteTool: Writes files asynchronously, tracks paths in session context, and reports session-aware metadata.
- SessionFileListTool: Lists files created/modified during the session, optionally filtered by pattern.

```mermaid
classDiagram
class SessionFileWriteTool {
+execute_impl(session_context, path, content, mode) ToolResult
}
class SessionFileListTool {
+execute_impl(session_context, filter_pattern) ToolResult
}
SessionFileWriteTool --> SessionFileListTool : "complementary"
```

**Diagram sources**
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L193)

**Section sources**
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L193)

## Dependency Analysis
- Coupling: HTTP handler depends on DB and Redis; gRPC service depends on Manager; Activities depend on Manager and pricing; OpenAI manager depends on Redis; tools depend on session context.
- Cohesion: Each component encapsulates a clear responsibility (persistence, API, orchestration, tooling).
- External dependencies: Redis for caching/persistence, PostgreSQL for durable metadata/history, pricing service for cost computation, LLM service for title generation.

```mermaid
graph LR
HTTP["HTTP Handler"] --> DB["PostgreSQL"]
HTTP --> Redis["Redis"]
GRPC["gRPC Service"] --> Manager["Session Manager"]
Manager --> Redis
Activities["Activities"] --> Manager
Activities --> Pricing["Pricing Service"]
OA["OpenAI Manager"] --> Redis
Tools["Session-aware Tools"] --> Manager
```

**Diagram sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L337)
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L185)
- [manager.go](file://go/orchestrator/internal/session/manager.go#L97-L283)
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L62-L133)
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L193)

**Section sources**
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L337)
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L185)
- [manager.go](file://go/orchestrator/internal/session/manager.go#L97-L283)
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L62-L133)
- [session_file.py](file://python/llm-service/llm_service/tools/builtin/session_file.py#L19-L193)

## Performance Considerations
- Redis caching: Local LRU cache reduces hot-path Redis calls; eviction trims half of the cache when exceeding max size. Monitor cache hits/misses and evictions.
- TTL alignment: Session JSON TTL matches ExpiresAt; ensure clock skew is acceptable; fallback to default TTL if negative.
- History bounds: Manager enforces max history; Activities also enforce a fixed upper bound when appending assistant messages.
- Indexing: Database indexes on external_id and non-deleted sessions improve query performance for dual-ID lookups and listing.
- Token aggregation: Prefer Redis for live token counts; fall back to DB aggregation for accuracy.
- Concurrency: Manager uses RWMutex around cache operations; minimize contention by avoiding frequent writes to the same session.

[No sources needed since this section provides general guidance]

## Troubleshooting Guide
Common issues and remedies:
- Session not found: Ensure correct session_id and tenant association; verify soft-deleted sessions are excluded; confirm dual-ID (external_id) usage.
- Session expired: Extend TTL via UpdateSession or CreateSession with explicit TTL; verify ExpiresAt and local cache eviction.
- Tenant isolation failure: Confirm auth context extraction and WHERE clauses in queries; ensure tenant_id matches.
- Token mismatch: Use Redis for live counts; if unavailable, aggregate from task_executions; verify pricing fallbacks.
- Title generation failures: LLM call failures trigger fallback truncation; check LLM service availability and environment overrides.
- OpenAI session collision: When provided session belongs to another user/tenant, a new session is created; inspect ResolveSession logs.

**Section sources**
- [manager.go](file://go/orchestrator/internal/session/manager.go#L186-L243)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L162-L180)
- [session.go](file://go/orchestrator/internal/activities/session_title.go#L96-L105)
- [session.go](file://go/orchestrator/cmd/gateway/internal/openai/session.go#L94-L112)

## Conclusion
The session subsystem provides robust, tenant-aware continuity with Redis-backed persistence, comprehensive token/cost tracking, and rich metadata/context handling. It supports multi-turn conversations, context switching, and resilient recovery via TTL extension and soft delete. The design balances performance with correctness, leveraging caching, indexing, and activity-driven updates.

[No sources needed since this section summarizes without analyzing specific files]

## Appendices

### Practical Examples

- Multi-turn conversation with context preservation
  - Create a session via gRPC or HTTP.
  - For each turn, call UpdateSessionResult to append assistant message and update totals.
  - Use GetSession to retrieve context and metrics; use GetSessionHistory for audit trails.

- Context switching across tasks
  - Use UpdateContext to switch focus (e.g., change topic or persona).
  - Maintain agent states via AgentState fields in session context.

- Session recovery after failures
  - On error, call GetSession to reload session; if expired, recreate with CreateSession or extend TTL via UpdateSession.
  - Use CleanupExpired periodically to remove stale sessions.

- Session recovery with external IDs
  - Use context->>'external_id' to link sessions to external systems; queries support both UUID and external_id.

- Soft deletion and cleanup
  - Soft delete sessions with deleted_at; queries filter by deleted_at IS NULL; periodic cleanup removes expired sessions.

**Section sources**
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L185)
- [session.go](file://go/orchestrator/internal/activities/session.go#L15-L142)
- [session.go](file://go/orchestrator/cmd/gateway/internal/handlers/session.go#L132-L337)
- [007_session_soft_delete.sql](file://migrations/postgres/007_session_soft_delete.sql#L7-L41)

### API and Protobuf Notes
- Session service RPCs: CreateSession, GetSession, UpdateSession, DeleteSession, AddMessage, ClearHistory.
- Protobuf fields: Session includes ID, user_id, timestamps, context, history, and metrics.

**Section sources**
- [session_service.go](file://go/orchestrator/internal/server/session_service.go#L34-L278)
- [session.proto](file://protos/session/session.proto)