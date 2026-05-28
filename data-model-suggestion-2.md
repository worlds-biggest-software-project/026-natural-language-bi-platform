# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Natural Language BI Platform · Created: 2026-05-11

## Philosophy

This model treats every meaningful action in the system as an immutable event. The single source of truth is an append-only event store; all queryable state (dashboards, conversations, query history) is derived by replaying or projecting events into materialized read models. The architecture follows the CQRS (Command Query Responsibility Segregation) pattern: writes go to the event store, reads come from purpose-built projections.

This approach is inspired by production event-sourced systems used in financial services, compliance platforms, and audit-critical SaaS applications. PostgreSQL serves as both the event store (using JSONB payloads with sequential ordering) and the projection database (materialized views and denormalized read tables). The pattern is well-documented in frameworks like Marten (.NET), EventStoreDB, and Axon Framework.

For a Natural Language BI platform, event sourcing is a natural fit because query audit trails are a core requirement — enterprises need to know exactly who asked what question, what SQL was generated, what data was returned, and whether any results were modified or overridden. An event-sourced model provides this auditability by design rather than as an afterthought.

**Best for:** Deployments where full audit trails are mandatory (regulated industries, financial data, healthcare analytics), where temporal queries are important ("what did this dashboard show last Tuesday?"), and where the system must support undo/replay of analytical workflows.

**Trade-offs:**
- (+) Complete, immutable audit trail of every user action — ideal for SOC 2, HIPAA, financial compliance
- (+) Temporal queries are trivial: reconstruct any entity's state at any point in time
- (+) Natural fit for undo/redo and versioned dashboards
- (+) Event replay enables rebuilding projections when requirements change (add a new read model without touching historical data)
- (+) Write path is simple and fast (append-only inserts)
- (-) Read path requires projections — more infrastructure to build and maintain
- (-) Higher storage requirements (events are never deleted, plus projections duplicate data)
- (-) Eventual consistency between event store and projections (typically milliseconds, but not zero)
- (-) Steeper learning curve for developers unfamiliar with event sourcing
- (-) Snapshot management needed for aggregates with long event histories (thousands of events per conversation)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CQRS / Event Sourcing (Martin Fowler, Greg Young) | Core architectural pattern; events as source of truth, projections as read models |
| CloudEvents v1.0.2 (CNCF) | Event envelope structure follows CloudEvents spec for interoperability |
| OpenTelemetry Semantic Conventions | Event metadata includes trace/span IDs for distributed tracing correlation |
| ISO 8601 | All event timestamps as TIMESTAMPTZ; event ordering by sequence number, not timestamp |
| SQL:2023 | Projections use standard PostgreSQL DDL; event store uses JSONB for payload flexibility |
| NIST SP 800-92 (Log Management) | Event retention and immutability practices align with NIST log management guidelines |

---

## 1. Event Store (Write Side)

### event_streams

```sql
-- Each aggregate (conversation, dashboard, data_source) has its own event stream
CREATE TABLE event_streams (
    stream_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL,              -- 'conversation', 'dashboard', 'data_source', 'semantic_model', 'user', 'organization'
    organization_id UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    current_version BIGINT NOT NULL DEFAULT 0   -- optimistic concurrency control
);

CREATE INDEX idx_event_streams_org ON event_streams (organization_id);
CREATE INDEX idx_event_streams_type ON event_streams (stream_type, organization_id);
```

### events

```sql
-- The immutable event store — append-only, never updated or deleted
CREATE TABLE events (
    sequence_num    BIGSERIAL PRIMARY KEY,      -- global ordering
    stream_id       UUID NOT NULL REFERENCES event_streams(stream_id),
    stream_version  BIGINT NOT NULL,            -- per-stream ordering (optimistic concurrency)
    event_type      TEXT NOT NULL,              -- e.g., 'QueryGenerated', 'DashboardCreated', 'VisualizationAdded'
    event_data      JSONB NOT NULL,             -- event payload (schema varies by event_type)
    metadata        JSONB NOT NULL DEFAULT '{}', -- causal_id, correlation_id, trace_id, user_agent
    organization_id UUID NOT NULL,
    user_id         UUID,                       -- NULL for system-generated events
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, stream_version)          -- prevents concurrent writes to same stream version
) PARTITION BY RANGE (created_at);

-- Partition by month for efficient time-range queries and retention management
CREATE TABLE events_2026_01 PARTITION OF events FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE events_2026_02 PARTITION OF events FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE events_2026_03 PARTITION OF events FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE events_2026_04 PARTITION OF events FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_2026_06 PARTITION OF events FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- ... continue monthly

CREATE INDEX idx_events_stream ON events (stream_id, stream_version);
CREATE INDEX idx_events_type ON events (event_type, created_at);
CREATE INDEX idx_events_org ON events (organization_id, created_at DESC);
CREATE INDEX idx_events_user ON events (user_id, created_at DESC) WHERE user_id IS NOT NULL;
```

### Event Type Catalog

The following event types define the domain vocabulary. Each has a specific JSONB payload schema:

```sql
-- Example event payloads by type:

-- ConversationStarted
-- {"conversation_id": "uuid", "title": null, "data_source_id": "uuid"}

-- MessageSent
-- {"conversation_id": "uuid", "message_id": "uuid", "role": "user", "content": "What was our revenue last quarter?"}

-- QueryGenerated
-- {"query_id": "uuid", "conversation_id": "uuid", "natural_language": "What was our revenue last quarter?",
--  "generated_sql": "SELECT SUM(amount) FROM orders WHERE created_at >= '2026-01-01'",
--  "sql_dialect": "postgresql", "confidence_score": 0.92, "llm_model": "claude-sonnet-4-20250514",
--  "semantic_refs": [{"model": "orders", "measure": "total_revenue", "dimensions": ["order_date"]}]}

-- QueryExecuted
-- {"query_id": "uuid", "status": "completed", "row_count": 1, "execution_time_ms": 234,
--  "result_hash": "sha256:abc123...", "columns": [...], "data": [...]}

-- QueryFailed
-- {"query_id": "uuid", "error_type": "syntax_error", "error_message": "column 'revenu' does not exist",
--  "generated_sql": "SELECT revenu FROM orders"}

-- QueryCorrected
-- {"query_id": "uuid", "original_sql": "...", "corrected_sql": "...", "correction_reason": "column_typo"}

-- VisualizationCreated
-- {"visualization_id": "uuid", "query_id": "uuid", "chart_type": "bar", "config": {...}}

-- DashboardCreated
-- {"dashboard_id": "uuid", "title": "Q1 Revenue Overview"}

-- DashboardVisualizationAdded
-- {"dashboard_id": "uuid", "visualization_id": "uuid", "position": {"x": 0, "y": 0, "w": 6, "h": 4}}

-- DashboardShared
-- {"dashboard_id": "uuid", "share_type": "link", "access_level": "view", "share_token": "..."}

-- DataSourceConnected
-- {"data_source_id": "uuid", "name": "Production DB", "engine": "postgresql"}

-- DataSourceSchemaSynced
-- {"data_source_id": "uuid", "tables_found": 42, "columns_found": 387}

-- SemanticModelPublished
-- {"semantic_model_id": "uuid", "name": "orders", "version": 3, "measures_count": 12, "dimensions_count": 8}

-- AlertTriggered
-- {"alert_id": "uuid", "condition_type": "threshold_above", "result_value": 15000, "threshold": 10000}

-- UserInvited
-- {"invited_email": "analyst@company.com", "role": "analyst", "invited_by": "uuid"}

-- PermissionGranted
-- {"user_id": "uuid", "role_id": "uuid", "data_source_id": "uuid", "access_level": "read"}
```

### snapshots

```sql
-- Periodic snapshots of aggregate state to avoid replaying long event histories
CREATE TABLE snapshots (
    stream_id       UUID NOT NULL REFERENCES event_streams(stream_id),
    stream_version  BIGINT NOT NULL,            -- version at which snapshot was taken
    snapshot_data   JSONB NOT NULL,             -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, stream_version)
);
```

---

## 2. Read Model Projections (Query Side)

Projections are denormalized read models built by processing events. They can be rebuilt from scratch by replaying the event store. Each projection has a tracking table to record its position in the event stream.

### projection_checkpoints

```sql
-- Tracks which events each projection has processed
CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,           -- 'conversations_projection', 'dashboards_projection', etc.
    last_sequence_num BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message   TEXT
);
```

### Projection: conversations_view

```sql
-- Materialized from: ConversationStarted, MessageSent, ConversationArchived, ConversationRenamed
CREATE TABLE conversations_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID NOT NULL,
    title           TEXT,
    message_count   INTEGER NOT NULL DEFAULT 0,
    last_message_at TIMESTAMPTZ,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_cv_user ON conversations_view (user_id, updated_at DESC);
CREATE INDEX idx_cv_org ON conversations_view (organization_id, updated_at DESC);
```

### Projection: messages_view

```sql
-- Materialized from: MessageSent
CREATE TABLE messages_view (
    id              UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    role            TEXT NOT NULL,
    content         TEXT NOT NULL,
    message_index   INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_mv_conversation ON messages_view (conversation_id, message_index);
```

### Projection: queries_view

```sql
-- Materialized from: QueryGenerated, QueryExecuted, QueryFailed, QueryCorrected
CREATE TABLE queries_view (
    id              UUID PRIMARY KEY,
    conversation_id UUID NOT NULL,
    message_id      UUID NOT NULL,
    user_id         UUID NOT NULL,
    organization_id UUID NOT NULL,
    data_source_id  UUID NOT NULL,
    natural_language TEXT NOT NULL,
    generated_sql   TEXT NOT NULL,
    final_sql       TEXT,                       -- after corrections
    sql_dialect     TEXT NOT NULL,
    status          TEXT NOT NULL,
    confidence_score NUMERIC(4,3),
    error_message   TEXT,
    row_count       INTEGER,
    execution_time_ms INTEGER,
    llm_model       TEXT,
    llm_tokens_used INTEGER,
    result_columns  JSONB,
    result_data     JSONB,
    result_hash     TEXT,
    correction_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_qv_conversation ON queries_view (conversation_id, created_at);
CREATE INDEX idx_qv_user ON queries_view (user_id, created_at DESC);
CREATE INDEX idx_qv_org ON queries_view (organization_id, created_at DESC);
CREATE INDEX idx_qv_hash ON queries_view (result_hash) WHERE result_hash IS NOT NULL;
```

### Projection: dashboards_view

```sql
-- Materialized from: DashboardCreated, DashboardUpdated, DashboardArchived,
--                     DashboardVisualizationAdded, DashboardVisualizationRemoved, DashboardShared
CREATE TABLE dashboards_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    visualization_count INTEGER NOT NULL DEFAULT 0,
    share_count     INTEGER NOT NULL DEFAULT 0,
    layout          JSONB NOT NULL DEFAULT '[]',
    filters         JSONB NOT NULL DEFAULT '[]',
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_dv_org ON dashboards_view (organization_id);
```

### Projection: visualizations_view

```sql
CREATE TABLE visualizations_view (
    id              UUID PRIMARY KEY,
    query_id        UUID NOT NULL,
    dashboard_id    UUID,                       -- NULL if not on a dashboard
    user_id         UUID NOT NULL,
    organization_id UUID NOT NULL,
    title           TEXT,
    chart_type      TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    insight_text    TEXT,
    is_saved        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_vv_org ON visualizations_view (organization_id) WHERE is_saved = TRUE;
CREATE INDEX idx_vv_dashboard ON visualizations_view (dashboard_id) WHERE dashboard_id IS NOT NULL;
```

### Projection: data_sources_view

```sql
-- Materialized from: DataSourceConnected, DataSourceUpdated, DataSourceSchemaSynced, DataSourceDisconnected
CREATE TABLE data_sources_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    table_count     INTEGER NOT NULL DEFAULT 0,
    column_count    INTEGER NOT NULL DEFAULT 0,
    last_synced_at  TIMESTAMPTZ,
    sync_status     TEXT NOT NULL DEFAULT 'pending',
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_dsv_org ON data_sources_view (organization_id);
```

### Projection: schema_metadata_view

```sql
-- Materialized from: DataSourceSchemaSynced (contains full table/column metadata in event payload)
CREATE TABLE schema_metadata_view (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL,
    schema_name     TEXT NOT NULL,
    table_name      TEXT NOT NULL,
    column_name     TEXT,                       -- NULL for table-level row
    data_type       TEXT,
    description     TEXT,
    is_primary_key  BOOLEAN DEFAULT FALSE,
    is_foreign_key  BOOLEAN DEFAULT FALSE,
    foreign_ref     TEXT,
    sample_values   TEXT[],
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,
    synced_at       TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_smv_source ON schema_metadata_view (data_source_id);
CREATE UNIQUE INDEX idx_smv_unique ON schema_metadata_view (data_source_id, schema_name, table_name, column_name);
```

### Projection: semantic_models_view

```sql
-- Materialized from: SemanticModelCreated, SemanticModelUpdated, SemanticModelPublished
CREATE TABLE semantic_models_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    data_source_id  UUID NOT NULL,
    name            TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    description     TEXT,
    source_type     TEXT NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 1,
    measures        JSONB NOT NULL DEFAULT '[]', -- denormalized array of measure definitions
    dimensions      JSONB NOT NULL DEFAULT '[]', -- denormalized array of dimension definitions
    joins           JSONB NOT NULL DEFAULT '[]', -- denormalized array of join definitions
    created_by      UUID NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_smv_org ON semantic_models_view (organization_id);
```

### Projection: alerts_view

```sql
CREATE TABLE alerts_view (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID NOT NULL,
    name            TEXT NOT NULL,
    description     TEXT,
    data_source_id  UUID NOT NULL,
    condition_type  TEXT NOT NULL,
    condition_value NUMERIC,
    schedule_cron   TEXT NOT NULL,
    notification_channels JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    trigger_count   INTEGER NOT NULL DEFAULT 0,
    last_evaluated_at TIMESTAMPTZ,
    last_triggered_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_av_org ON alerts_view (organization_id);
CREATE INDEX idx_av_active ON alerts_view (is_active) WHERE is_active = TRUE;
```

### Projection: query_analytics_view

```sql
-- Denormalized analytical projection for usage analytics and performance monitoring
-- Materialized from: QueryGenerated, QueryExecuted, QueryFailed
CREATE TABLE query_analytics_view (
    query_id        UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID NOT NULL,
    data_source_id  UUID NOT NULL,
    query_date      DATE NOT NULL,
    hour_of_day     SMALLINT NOT NULL,
    status          TEXT NOT NULL,
    confidence_score NUMERIC(4,3),
    execution_time_ms INTEGER,
    llm_model       TEXT,
    llm_tokens_used INTEGER,
    semantic_models_used TEXT[],
    measures_used   TEXT[],
    dimensions_used TEXT[],
    table_count     SMALLINT,
    join_count      SMALLINT,
    was_corrected   BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (query_date);

-- Partition by month for analytics
CREATE TABLE qa_2026_01 PARTITION OF query_analytics_view FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE qa_2026_02 PARTITION OF query_analytics_view FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... continue monthly

CREATE INDEX idx_qav_org_date ON query_analytics_view (organization_id, query_date DESC);
CREATE INDEX idx_qav_user_date ON query_analytics_view (user_id, query_date DESC);
CREATE INDEX idx_qav_model ON query_analytics_view (llm_model, query_date);
```

---

## 3. Configuration Tables (Non-Event-Sourced)

Some configuration data is better stored as traditional mutable state rather than event-sourced, because it is reference data that does not benefit from temporal querying.

### organizations

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL DEFAULT 'local',
    auth_subject    TEXT,
    password_hash   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);
```

### roles

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);
```

### user_roles

```sql
CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

### llm_providers

```sql
CREATE TABLE llm_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    provider_type   TEXT NOT NULL,
    api_key_encrypted TEXT NOT NULL,
    default_model   TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);
```

### data_source_credentials

```sql
-- Separated from events for security — connection secrets are never written to the event store
CREATE TABLE data_source_credentials (
    data_source_id  UUID PRIMARY KEY,
    connection_config_encrypted JSONB NOT NULL, -- AES-256 encrypted connection parameters
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example: Temporal Query (Reconstructing Past State)

One of the key benefits of event sourcing is reconstructing state at any point in time:

```sql
-- Reconstruct a dashboard's state as of a specific date
-- "What did the Q1 Revenue dashboard look like on March 15?"
WITH dashboard_events AS (
    SELECT
        e.event_type,
        e.event_data,
        e.created_at,
        e.stream_version
    FROM events e
    JOIN event_streams es ON e.stream_id = es.stream_id
    WHERE es.stream_type = 'dashboard'
      AND e.event_data->>'dashboard_id' = '550e8400-e29b-41d4-a716-446655440000'
      AND e.created_at <= '2026-03-15 23:59:59Z'
    ORDER BY e.stream_version ASC
)
SELECT * FROM dashboard_events;

-- The application replays these events in order to reconstruct the dashboard state
-- as it existed on March 15, including which visualizations were present,
-- what the layout was, and who had access.
```

## Example: Audit Query (Who Changed What?)

```sql
-- Full audit trail for a specific conversation
SELECT
    e.sequence_num,
    e.event_type,
    u.display_name AS actor,
    e.event_data,
    e.created_at
FROM events e
JOIN event_streams es ON e.stream_id = es.stream_id
LEFT JOIN users u ON e.user_id = u.id
WHERE es.stream_type = 'conversation'
  AND e.event_data->>'conversation_id' = '550e8400-e29b-41d4-a716-446655440001'
ORDER BY e.sequence_num ASC;
```

## Example: Analytics Over Event Data

```sql
-- Query accuracy trend over the past 30 days
SELECT
    DATE(created_at) AS query_date,
    COUNT(*) AS total_queries,
    COUNT(*) FILTER (WHERE event_type = 'QueryExecuted') AS successful,
    COUNT(*) FILTER (WHERE event_type = 'QueryFailed') AS failed,
    COUNT(*) FILTER (WHERE event_type = 'QueryCorrected') AS corrected,
    AVG((event_data->>'confidence_score')::NUMERIC)
        FILTER (WHERE event_type = 'QueryGenerated') AS avg_confidence
FROM events
WHERE event_type IN ('QueryGenerated', 'QueryExecuted', 'QueryFailed', 'QueryCorrected')
  AND created_at >= now() - INTERVAL '30 days'
GROUP BY DATE(created_at)
ORDER BY query_date DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (Write Side) | 3 | event_streams, events (partitioned), snapshots |
| Read Projections | 11 | conversations, messages, queries, dashboards, visualizations, data_sources, schema_metadata, semantic_models, alerts, query_analytics (partitioned), projection_checkpoints |
| Configuration (Mutable) | 6 | organizations, users, roles, user_roles, llm_providers, data_source_credentials |
| **Total** | **20** | Plus monthly partitions for events and query_analytics |

---

## Key Design Decisions

1. **Events are the single source of truth.** All user actions — asking questions, generating SQL, creating dashboards, sharing results — are captured as immutable events. The `events` table is append-only; rows are never updated or deleted. This guarantees a complete, tamper-proof audit trail.

2. **Projections are disposable and rebuildable.** Every `_view` table can be dropped and rebuilt by replaying events from the event store. This means schema changes to read models are cheap: add a column to a projection, replay events to populate it, no data migration needed.

3. **Credentials are never event-sourced.** Connection strings, API keys, and passwords are stored in separate, mutable configuration tables (`data_source_credentials`, `llm_providers`). Writing secrets to an immutable event store would make them irrevocable, which is a security anti-pattern.

4. **Events are partitioned by month.** The `events` table is range-partitioned on `created_at`, enabling efficient time-range queries and making retention management straightforward (drop old partitions). PostgreSQL 14+ supports partition pruning automatically.

5. **Semantic models are denormalized in projections.** Unlike the normalized model (Suggestion 1), semantic measures, dimensions, and joins are stored as JSONB arrays within the `semantic_models_view` projection rather than separate tables. This simplifies the read path for the NL-to-SQL engine, which needs the full model definition in a single fetch.

6. **Query analytics are a dedicated projection.** The `query_analytics_view` is a purpose-built, time-partitioned projection optimized for operational analytics: query success rates, LLM token usage, execution times, confidence score trends. This separation follows the CQRS principle of building read models tailored to specific query patterns.

7. **Optimistic concurrency via stream versioning.** Each event stream has a `current_version` counter, and each event records its `stream_version`. Writers must specify the expected version when appending, preventing lost updates when multiple users modify the same aggregate concurrently.

8. **CloudEvents-compatible metadata.** The `metadata` JSONB field on each event follows CloudEvents v1.0.2 conventions (`correlation_id`, `causation_id`, `trace_id`), enabling integration with distributed tracing systems and event mesh architectures.
