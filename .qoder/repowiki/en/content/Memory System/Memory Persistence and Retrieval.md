# Memory Persistence and Retrieval

<cite>
**Referenced Files in This Document**
- [agent_memory.go](file://go/orchestrator/internal/activities/agent_memory.go)
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go)
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go)
- [001_initial_schema.sql](file://migrations/postgres/001_initial_schema.sql)
- [002_persistence_tables.sql](file://migrations/postgres/002_persistence_tables.sql)
- [create_collections.py](file://migrations/qdrant/create_collections.py)
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
This document explains how Shannon persists and retrieves conversational history, agent memories, and contextual data across sessions and system restarts. It covers the dual-layer persistence architecture: PostgreSQL for structured execution and audit data, and Qdrant for vectorized semantic memory and fast similarity search. It also documents database schema, indexing strategies, data lifecycle management, session continuity, backup and restore procedures, data migration strategies, archival policies, consistency guarantees, and performance optimizations for large-scale memory storage.

## Project Structure
The memory system spans three primary areas:
- Vector database (Qdrant) for semantic memory and similarity search
- Relational database (PostgreSQL) for structured execution, auditing, and analytics
- Activity layer that orchestrates memory reads/writes across sessions and agents

```mermaid
graph TB
subgraph "Vector DB (Qdrant)"
VClient["vectordb.Client<br/>collections: task_embeddings, summaries,<br/>tool_results, cases, document_chunks"]
VFilters["Payload filters:<br/>session_id, tenant_id, user_id, agent_id,<br/>qa_id, is_chunked, timestamp"]
end
subgraph "Relational DB (PostgreSQL)"
PClient["db.Client<br/>async write queue, workers"]
PTables["Core tables:<br/>task_executions, agent_executions, tool_executions,<br/>session_archives, usage_daily_aggregates"]
end
subgraph "Activities"
ASession["FetchSessionMemory"]
AAgent["FetchAgentMemory"]
ASem["FetchSemanticMemory / Chunked"]
AHier["FetchHierarchicalMemory"]
end
ASession --> VClient
AAgent --> VClient
ASem --> VClient
AHier --> VClient
AHier --> ASession
AHier --> ASem
ASession --> PClient
AAgent --> PClient
ASem --> PClient
AHier --> PClient
VClient --> VFilters
```

**Diagram sources**
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L1-L439)
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L1-L68)
- [agent_memory.go](file://go/orchestrator/internal/activities/agent_memory.go#L1-L89)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L1-L222)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L1-L351)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go#L1-L245)

**Section sources**
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L1-L439)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go#L1-L245)

## Core Components
- Vector DB client: Provides semantic search, upsert, and collection management for embeddings and summaries.
- Activity layer: Implements fetching and recording of session, agent, and semantic memory with deduplication and chunk aggregation.
- Relational DB client: Manages asynchronous writes for task, agent, tool, session archive, and audit logs with batching and health checks.
- Database schema: Defines tables and indexes for structured persistence and analytics.

Key responsibilities:
- Persist Q&A pairs and summaries as embeddings with session/tenant/user/agent filters.
- Retrieve recent and semantic context with optional diversity re-ranking.
- Archive session snapshots and maintain daily usage aggregates.
- Enforce graceful degradation and circuit-breaking for resilience.

**Section sources**
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L1-L439)
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L1-L68)
- [agent_memory.go](file://go/orchestrator/internal/activities/agent_memory.go#L1-L89)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L1-L222)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L1-L351)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go#L1-L245)

## Architecture Overview
The memory architecture integrates vector and relational persistence with Temporal activities:

```mermaid
sequenceDiagram
participant WF as "Temporal Workflow"
participant ACT as "Activity Layer"
participant VDB as "Qdrant Client"
participant PG as "PostgreSQL Client"
WF->>ACT : "FetchSessionMemory(session_id, tenant_id, top_k)"
ACT->>VDB : "GetSessionContext(session_id, tenant_id, top_k)"
VDB-->>ACT : "[]ContextItem"
ACT-->>WF : "Items"
WF->>ACT : "FetchAgentMemory(session_id, agent_id, tenant_id, top_k)"
ACT->>VDB : "GetAgentContext(...)"
VDB-->>ACT : "[]ContextItem"
ACT-->>WF : "Items"
WF->>ACT : "FetchSemanticMemory(query, session_id, threshold, top_k)"
ACT->>VDB : "GetSessionContextSemanticByEmbedding(embedding, ...)"
VDB-->>ACT : "[]ContextItem"
ACT-->>WF : "Items"
WF->>ACT : "FetchHierarchicalMemory(query, session_id, ...)"
ACT->>ACT : "Combine recent + semantic + summaries"
ACT-->>WF : "Deduplicated Items"
WF->>ACT : "RecordAgentMemory(...)"
ACT->>PG : "Async write (queue)"
ACT->>VDB : "UpsertTaskEmbedding(...)"
VDB-->>ACT : "OK"
ACT-->>WF : "Result"
```

**Diagram sources**
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L1-L68)
- [agent_memory.go](file://go/orchestrator/internal/activities/agent_memory.go#L1-L89)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L1-L222)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L1-L351)
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L1-L439)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)

## Detailed Component Analysis

### Vector Memory Retrieval and Storage
- Session memory: Retrieves recent items scoped to a session and tenant, emitting progress events and metrics.
- Agent memory: Filters by both session and agent identifiers for agent-scoped context.
- Semantic memory: Generates embeddings for queries and performs similarity search with optional MMR diversity re-ranking and chunk aggregation.
- Hierarchical memory: Combines recent, semantic, and summary results with deduplication across sources.

```mermaid
flowchart TD
Start(["FetchHierarchicalMemory"]) --> Recent["FetchSessionMemory(recent_top_k)"]
Recent --> MetricsRecent["Record metrics (recent)"]
MetricsRecent --> MergeRecent["Add items with _source='recent'"]
Start --> Semantic["FetchSemanticMemoryChunked(semantic_top_k, threshold)"]
Semantic --> DedupBuild["Build seen set from recent items"]
DedupBuild --> MergeSemantic["Add non-duplicate items with _source='semantic'"]
Start --> Summaries["SearchSummaries(summary_top_k)"]
Summaries --> MergeSummaries["Add non-duplicate items with _source='summary'"]
MergeRecent --> Limit["Limit total items"]
MergeSemantic --> Limit
MergeSummaries --> Limit
Limit --> End(["Return Items + Sources"])
```

**Diagram sources**
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L50-L222)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L16-L276)
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L26-L67)

**Section sources**
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L1-L68)
- [agent_memory.go](file://go/orchestrator/internal/activities/agent_memory.go#L1-L89)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L1-L222)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L1-L351)
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L232-L438)

### Relational Persistence and Async Writes
- Asynchronous write queue with configurable workers batches writes for task, agent, tool, session archive, and audit logs.
- Health checks and circuit-breaker protection for database operations.
- Structured models define JSONB fields for flexible payloads and metadata.

```mermaid
classDiagram
class DBClient {
+Config
+QueueWrite(type, data, callback)
+QueueWriteWithRetry(...)
+WithTransactionCB(ctx, fn)
+Close()
}
class TaskExecution {
+UUID id
+string workflow_id
+UUID user_id?
+string session_id
+string query
+string status
+JSONB metadata
+int total_tokens
+float total_cost_usd
}
class AgentExecution {
+UUID id
+UUID task_execution_id
+string agent_id
+string input
+string output
+string state
+int tokens_used
+JSONB metadata
}
class ToolExecution {
+UUID id
+UUID agent_execution_id
+UUID task_execution_id
+string tool_name
+JSONB input_params
+JSONB output
+bool success
+int duration_ms
+int tokens_consumed
+JSONB metadata
}
class SessionArchive {
+UUID id
+string session_id
+UUID user_id?
+JSONB snapshot_data
+int message_count
+int total_tokens
+float total_cost_usd
+time session_started_at
+time snapshot_taken_at
+time ttl_expires_at?
}
DBClient --> TaskExecution : "writes"
DBClient --> AgentExecution : "writes"
DBClient --> ToolExecution : "writes"
DBClient --> SessionArchive : "writes"
```

**Diagram sources**
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go#L62-L170)

**Section sources**
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go#L1-L245)

### Database Schema and Indexing Strategies
- Initial schema establishes users, sessions, prompts, learning cases, token usage, and audit logs with appropriate indexes.
- Persistence tables add task_executions, agent_executions, tool_executions, session_archives, and usage_daily_aggregates with foreign keys and indexes.
- Qdrant collections include task_embeddings, tool_results, cases, document_chunks, and summaries with payload indexes for session/tenant/user/agent and temporal fields.

```mermaid
erDiagram
USERS {
uuid id PK
string external_id UK
string email
uuid tenant_id
jsonb metadata
timestamptz created_at
timestamptz updated_at
}
SESSIONS {
uuid id PK
uuid user_id FK
jsonb context
int token_budget
int tokens_used
timestamptz created_at
timestamptz updated_at
timestamptz expires_at
}
TASK_EXECUTIONS {
uuid id PK
string workflow_id UK
uuid user_id FK
string session_id
text query
string mode
string status
timestamptz started_at
timestamptz completed_at
text result
text error_message
int total_tokens
int prompt_tokens
int completion_tokens
float total_cost_usd
int duration_ms
int agents_used
int tools_invoked
int cache_hits
decimal complexity_score
jsonb metadata
timestamptz created_at
}
AGENT_EXECUTIONS {
uuid id PK
uuid task_execution_id FK
string agent_id
int execution_order
text input
text output
string mode
string state
int tokens_used
float cost_usd
string model_used
int duration_ms
int memory_used_mb
timestamptz created_at
timestamptz completed_at
}
TOOL_EXECUTIONS {
uuid id PK
uuid agent_execution_id FK
uuid task_execution_id FK
string tool_name
string tool_version
string category
jsonb input_params
jsonb output
bool success
text error_message
int duration_ms
int tokens_consumed
bool sandboxed
int memory_used_mb
timestamptz executed_at
}
SESSION_ARCHIVES {
uuid id PK
string session_id
uuid user_id FK
jsonb snapshot_data
int message_count
int total_tokens
float total_cost_usd
timestamptz session_started_at
timestamptz snapshot_taken_at
timestamptz ttl_expires_at
}
USERS ||--o{ SESSIONS : "owns"
USERS ||--o{ TASK_EXECUTIONS : "involved_in"
TASK_EXECUTIONS ||--o{ AGENT_EXECUTIONS : "contains"
AGENT_EXECUTIONS ||--o{ TOOL_EXECUTIONS : "invokes"
USERS ||--o{ SESSION_ARCHIVES : "archived_by"
```

**Diagram sources**
- [001_initial_schema.sql](file://migrations/postgres/001_initial_schema.sql#L9-L141)
- [002_persistence_tables.sql](file://migrations/postgres/002_persistence_tables.sql#L8-L254)

**Section sources**
- [001_initial_schema.sql](file://migrations/postgres/001_initial_schema.sql#L1-L141)
- [002_persistence_tables.sql](file://migrations/postgres/002_persistence_tables.sql#L1-L254)

### Session Continuity and Data Lifecycle
- Session continuity: Activities filter by session_id and tenant_id to maintain coherent context across agent interactions and workflow executions.
- Data lifecycle: Qdrant payloads include timestamps and identifiers enabling recent-first retrieval and deduplication. PostgreSQL tables track execution history, usage, and audits for lifecycle management and analytics.

**Section sources**
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L26-L67)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L50-L222)
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L232-L438)
- [models.go (PostgreSQL)](file://go/orchestrator/internal/db/models.go#L62-L170)

### Practical Examples: Backup, Restore, Migration, Archival
- Backup and restore:
  - PostgreSQL: Use logical backups (e.g., pg_dump/pg_restore) for task_executions, agent_executions, tool_executions, session_archives, and audit_logs.
  - Qdrant: Back up collection data externally (e.g., export points) and recreate collections with payload indexes.
- Data migration:
  - Apply SQL migrations in order; ensure foreign keys and indexes are created as per migration scripts.
  - For vector data, re-index or re-upsert points after schema changes.
- Archival:
  - Periodic snapshots of session state are stored in session_archives for long-term retention and offline analysis.

**Section sources**
- [002_persistence_tables.sql](file://migrations/postgres/002_persistence_tables.sql#L112-L134)
- [create_collections.py](file://migrations/qdrant/create_collections.py#L44-L227)

### Relationship Between In-Memory Caching and Persistent Storage
- In-memory caching: Not explicitly implemented in the referenced code; vector and relational persistence are the primary stores.
- Persistent storage: Qdrant for embeddings and summaries; PostgreSQL for structured execution and audit data.
- Consistency: Activities gracefully degrade when services are unavailable; async writes ensure durability without blocking workflows.
- Eventual consistency: Vector search results may lag behind latest upserts by refresh intervals; deduplication mitigates stale overlaps.

**Section sources**
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L96-L169)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L148-L196)

### Data Retention, GDPR Compliance, Secure Deletion
- Retention: Configure TTL on session_archives and Qdrant payload timestamps; implement soft-delete patterns for sessions.
- GDPR: Provide user-centric deletion by filtering and removing records by user_id, session_id, and tenant_id; redact PII during ingestion when requested.
- Secure deletion: Remove vectors and associated payloads from Qdrant; cascade deletes for relational tables; purge audit trails per policy.

[No sources needed since this section provides general guidance]

## Dependency Analysis
The activity layer depends on the vector DB client for semantic and recent context retrieval and on the relational DB client for durable writes. The vector DB client encapsulates HTTP calls to Qdrant with circuit breaking and tracing.

```mermaid
graph LR
ASession["FetchSessionMemory"] --> VClient["vectordb.Client"]
AAgent["FetchAgentMemory"] --> VClient
ASem["FetchSemanticMemory / Chunked"] --> VClient
AHier["FetchHierarchicalMemory"] --> VClient
AHier --> ASession
AHier --> ASem
ASession --> PClient["db.Client"]
AAgent --> PClient
ASem --> PClient
AHier --> PClient
```

**Diagram sources**
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L1-L68)
- [agent_memory.go](file://go/orchestrator/internal/activities/agent_memory.go#L1-L89)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L1-L222)
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L1-L351)
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L1-L439)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)

**Section sources**
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L1-L439)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L1-L498)

## Performance Considerations
- Vector search:
  - Use MMR re-ranking for diversity when enabled; tune lambda and pool multiplier.
  - Index payload fields in Qdrant (session_id, tenant_id, user_id, agent_id, qa_id, is_chunked, timestamp).
- Relational writes:
  - Async write queue with configurable workers and batch flushing improves throughput.
  - Health checks and circuit breakers protect against overload.
- Query patterns:
  - Favor session-scoped filters to reduce search space.
  - Limit returned items to prevent context overflow.

**Section sources**
- [semantic_memory_chunked.go](file://go/orchestrator/internal/activities/semantic_memory_chunked.go#L85-L97)
- [client.go (vector DB)](file://go/orchestrator/internal/vectordb/client.go#L114-L169)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L148-L196)

## Troubleshooting Guide
- Vector DB unavailability:
  - Activities gracefully return empty results; metrics reflect misses.
  - Circuit breaker protects downstream calls; inspect Qdrant connectivity and payload indexes.
- Database queue saturation:
  - Synchronous fallback occurs when queue is full; monitor worker counts and batch sizes.
  - Health checks log failures; investigate connection pool limits and SSL settings.
- Deduplication anomalies:
  - Ensure _point_id and composite keys are populated; verify payload structure consistency.

**Section sources**
- [session_memory.go](file://go/orchestrator/internal/activities/session_memory.go#L32-L46)
- [semantic_memory.go](file://go/orchestrator/internal/activities/semantic_memory.go#L84-L162)
- [client.go (PostgreSQL)](file://go/orchestrator/internal/db/client.go#L332-L391)

## Conclusion
Shannon’s memory system combines Qdrant vector search with PostgreSQL relational persistence to deliver robust, scalable, and resilient conversational memory. Activities orchestrate session, agent, and semantic retrieval with deduplication and chunk aggregation, while async writes and health checks ensure durability and performance. Proper indexing, lifecycle management, and GDPR-aligned deletion enable secure and compliant operations at scale.

## Appendices

### Appendix A: Vector Collections and Payload Indexes
- Collections: task_embeddings, tool_results, cases, document_chunks, summaries.
- Payload indexes: session_id, tenant_id, user_id, agent_id, qa_id, is_chunked, timestamp, reward, tool_name, document_id, chunk_index.

**Section sources**
- [create_collections.py](file://migrations/qdrant/create_collections.py#L44-L227)