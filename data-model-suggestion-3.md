# Data Model Suggestion 3: Hybrid Relational + JSONB with Vector Embeddings

> Project: Natural Language BI Platform · Created: 2026-05-11

## Philosophy

This model combines three storage strategies within a single PostgreSQL database: normalized relational tables for core entities with well-defined structures, JSONB columns for variable/extensible metadata that differs across connectors, chart types, and deployment contexts, and pgvector embeddings for semantic search over schema metadata — the critical ingredient for RAG-powered NL-to-SQL accuracy.

The key insight is that a Natural Language BI platform's most important differentiator is query accuracy, and accuracy depends on feeding the LLM the right schema context for each question. Rather than sending the entire database schema to the LLM (which exceeds context windows for large databases), this model stores vector embeddings of table descriptions, column descriptions, and metric definitions alongside the relational data. At query time, the user's natural language question is embedded and compared against the stored vectors to retrieve only the most relevant schema context — a RAG (Retrieval-Augmented Generation) pipeline built directly into the application database.

This approach is inspired by production RAG systems using PostgreSQL + pgvector, the schema metadata patterns documented in text-to-SQL research (DIN-SQL, DAIL-SQL), and the JSONB hybrid patterns used by platforms like Supabase and Hasura where schema flexibility is needed without sacrificing query performance on core fields.

**Best for:** Teams building an MVP that will evolve rapidly (JSONB handles schema changes without migrations), deployments connecting to large enterprise databases with hundreds of tables (vector search scales where brute-force schema context does not), and architectures where the NL-to-SQL accuracy pipeline is the primary competitive advantage.

**Trade-offs:**
- (+) Vector embeddings enable semantic schema retrieval — the most impactful accuracy improvement for NL-to-SQL
- (+) JSONB columns eliminate migrations for connector-specific or chart-type-specific configuration
- (+) Fewer tables than normalized model (~22 vs ~27) while supporting more flexibility
- (+) Single database (PostgreSQL + pgvector) — no external vector database needed
- (+) GIN indexes on JSONB enable efficient queries on variable fields
- (-) JSONB fields lack referential integrity — application must validate structure
- (-) pgvector adds a PostgreSQL extension dependency (though widely available on managed PostgreSQL services)
- (-) Embedding generation requires an additional API call per schema element (amortized during sync, not at query time)
- (-) JSONB query performance degrades for deeply nested structures without careful indexing
- (-) Developers must understand both relational and document query patterns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| pgvector (PostgreSQL Extension) | Stores and indexes vector embeddings for semantic schema retrieval |
| HNSW Index Algorithm | Used for approximate nearest neighbor search on embeddings (faster than IVFFlat for write-heavy workloads) |
| OpenAI / Anthropic Embedding APIs | Embedding dimensions (1536 for text-embedding-3-small, 1024 for voyage-3) define the vector column size |
| Cube.js Data Model Spec | Semantic model structure stored as JSONB matching Cube.js YAML schema for bidirectional sync |
| dbt Semantic Layer (MetricFlow) | Entity/dimension/metric JSONB structures align with MetricFlow types |
| JSON Schema (Draft 2020-12) | JSONB columns documented with JSON Schema for application-level validation |
| ISO 8601 | All timestamps as TIMESTAMPTZ |
| GIN Indexing (PostgreSQL) | JSONB columns indexed with GIN operators for containment and key-exists queries |

---

## 0. Extensions

```sql
-- Required PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";     -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "vector";       -- pgvector for embeddings
```

---

## 1. Identity & Access Control

### organizations

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan            TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {"default_llm_provider": "anthropic", "default_model": "claude-sonnet-4-20250514",
    --  "embedding_model": "text-embedding-3-small", "embedding_dimensions": 1536,
    --  "max_result_rows": 10000, "query_timeout_seconds": 30,
    --  "features": {"self_healing_dashboards": true, "proactive_insights": false}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth            JSONB NOT NULL DEFAULT '{}',
    -- Example auth:
    -- {"provider": "google", "subject": "google-oauth2|12345",
    --  "claims": {"email_verified": true, "hd": "company.com"}}
    -- For local auth:
    -- {"provider": "local", "password_hash": "$2b$12$..."}
    roles           TEXT[] NOT NULL DEFAULT '{viewer}',  -- denormalized role list for fast checks
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example preferences:
    -- {"theme": "dark", "default_chart_type": "bar", "timezone": "America/New_York",
    --  "notification_channels": ["email", "slack"]}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
CREATE INDEX idx_users_roles ON users USING GIN (roles);
```

### roles

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- [{"resource": "data_source", "actions": ["read", "connect", "sync"]},
    --  {"resource": "dashboard", "actions": ["read", "create", "share"]},
    --  {"resource": "query", "actions": ["read", "execute"]},
    --  {"resource": "semantic_model", "actions": ["read"]}]
    is_system       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_roles_org ON roles (organization_id);
```

### user_roles

```sql
CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

---

## 2. Data Sources & Schema Metadata with Embeddings

### data_sources

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL,              -- 'postgresql', 'snowflake', 'bigquery', etc.
    connection      JSONB NOT NULL,             -- encrypted, engine-specific connection params
    -- Example for PostgreSQL:
    -- {"host": "db.example.com", "port": 5432, "database": "analytics",
    --  "username": "readonly", "ssl_mode": "require", "password_encrypted": "..."}
    -- Example for Snowflake:
    -- {"account": "xy12345.us-east-1", "warehouse": "COMPUTE_WH",
    --  "database": "ANALYTICS", "schema": "PUBLIC", "role": "ANALYST",
    --  "auth_type": "key_pair", "private_key_encrypted": "..."}
    -- Example for BigQuery:
    -- {"project_id": "my-project", "dataset": "analytics",
    --  "credentials_encrypted": "..."}
    sync_state      JSONB NOT NULL DEFAULT '{"status": "pending"}',
    -- Example: {"status": "synced", "last_synced_at": "2026-05-11T10:30:00Z",
    --           "table_count": 42, "column_count": 387, "error": null}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources (organization_id);
```

### schema_elements

```sql
-- Unified table for both tables and columns, with vector embeddings for RAG
CREATE TABLE schema_elements (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    element_type    TEXT NOT NULL CHECK (element_type IN ('table', 'column')),
    schema_name     TEXT NOT NULL,              -- e.g., 'public', 'analytics'
    table_name      TEXT NOT NULL,
    column_name     TEXT,                       -- NULL for table-level elements
    data_type       TEXT,                       -- NULL for table-level elements
    description     TEXT,                       -- human or LLM-generated description
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- For tables:
    -- {"row_count_estimate": 1500000, "table_type": "table",
    --  "primary_key": ["id"], "foreign_keys": [{"column": "user_id", "references": "users.id"}]}
    -- For columns:
    -- {"is_nullable": true, "is_primary_key": false, "is_foreign_key": true,
    --  "foreign_table": "users", "foreign_column": "id", "ordinal_position": 3,
    --  "sample_values": ["US", "EU", "APAC"], "distinct_count": 12}
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,

    -- Vector embedding of the description + context for semantic retrieval
    -- Description is embedded as: "{schema}.{table}.{column}: {data_type} - {description}"
    -- e.g., "public.orders.total_amount: numeric - The total monetary value of the order in USD"
    embedding       vector(1536),              -- matches text-embedding-3-small dimensions

    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schema_elements_source ON schema_elements (data_source_id, element_type);
CREATE UNIQUE INDEX idx_schema_elements_unique ON schema_elements (
    data_source_id, schema_name, table_name, COALESCE(column_name, '__TABLE__')
);

-- HNSW index for fast approximate nearest neighbor search on embeddings
CREATE INDEX idx_schema_elements_embedding ON schema_elements
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### Example: Semantic Schema Retrieval for NL-to-SQL

```sql
-- When a user asks "What was our revenue by region last quarter?", the system:
-- 1. Embeds the question using the same embedding model
-- 2. Finds the most relevant schema elements via vector similarity
-- 3. Passes only those elements as context to the LLM

-- Step 2: Retrieve top-10 most relevant schema elements
SELECT
    se.schema_name,
    se.table_name,
    se.column_name,
    se.data_type,
    se.description,
    se.metadata,
    1 - (se.embedding <=> $1::vector) AS similarity  -- $1 = embedded user question
FROM schema_elements se
WHERE se.data_source_id = $2                          -- $2 = selected data source
  AND se.is_visible = TRUE
  AND 1 - (se.embedding <=> $1::vector) > 0.3        -- minimum similarity threshold
ORDER BY se.embedding <=> $1::vector
LIMIT 10;

-- This might return:
-- public.orders.total_amount (numeric) - "Total order value in USD"       similarity: 0.87
-- public.orders.region (text) - "Geographic sales region"                 similarity: 0.85
-- public.orders.created_at (timestamptz) - "When the order was placed"    similarity: 0.82
-- public.customers.region (text) - "Customer's home region"               similarity: 0.79
-- public.revenue_summary (table) - "Monthly revenue aggregation table"    similarity: 0.76
-- ... etc.
```

---

## 3. Semantic Layer

### semantic_models

```sql
CREATE TABLE semantic_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    description     TEXT,
    source_type     TEXT NOT NULL CHECK (source_type IN ('cubejs', 'dbt', 'custom')),
    definition      JSONB NOT NULL,
    -- For Cube.js source:
    -- {"sql_table": "public.orders",
    --  "measures": [
    --    {"name": "total_revenue", "display_name": "Total Revenue", "type": "sum",
    --     "sql": "total_amount", "format": "$#,##0.00",
    --     "description": "Sum of all order amounts in USD"},
    --    {"name": "order_count", "display_name": "Order Count", "type": "count",
    --     "sql": "id", "description": "Total number of orders"}
    --  ],
    --  "dimensions": [
    --    {"name": "region", "display_name": "Region", "type": "string",
    --     "sql": "region", "description": "Geographic sales region"},
    --    {"name": "order_date", "display_name": "Order Date", "type": "time",
    --     "sql": "created_at", "granularity": ["day", "week", "month", "quarter", "year"]}
    --  ],
    --  "joins": [
    --    {"to": "customers", "relationship": "many_to_one",
    --     "sql": "{orders}.customer_id = {customers}.id"}
    --  ]}
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 1,

    -- Embedding of the full semantic model description for RAG
    -- Includes model name, description, all measure/dimension names and descriptions
    embedding       vector(1536),

    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_semantic_models_org ON semantic_models (organization_id);
CREATE INDEX idx_semantic_models_definition ON semantic_models USING GIN (definition);
CREATE INDEX idx_semantic_models_embedding ON semantic_models
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### semantic_glossary

```sql
-- Business term glossary for disambiguation
-- Maps business jargon to specific measures/dimensions
CREATE TABLE semantic_glossary (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    term            TEXT NOT NULL,              -- e.g., 'ARR', 'churn rate', 'MQL'
    definition      TEXT NOT NULL,              -- business definition
    synonyms        TEXT[] NOT NULL DEFAULT '{}', -- alternative terms
    maps_to         JSONB NOT NULL DEFAULT '{}',
    -- Example: {"semantic_model": "subscriptions", "measure": "annual_recurring_revenue"}
    -- Or: {"semantic_model": "customers", "measure": "churn_rate",
    --       "default_filters": {"period": "trailing_12_months"}}
    embedding       vector(1536),              -- embedded term + definition + synonyms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, term)
);

CREATE INDEX idx_glossary_org ON semantic_glossary (organization_id);
CREATE INDEX idx_glossary_synonyms ON semantic_glossary USING GIN (synonyms);
CREATE INDEX idx_glossary_embedding ON semantic_glossary
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

### Example: Combined Semantic Search for NL Context Assembly

```sql
-- When a user asks "Show me ARR by region for enterprise customers",
-- the system searches across schema elements, semantic models, AND the glossary
-- to assemble the most relevant context for the LLM.

WITH question_embedding AS (
    SELECT $1::vector AS emb  -- $1 = embedded user question
),
schema_matches AS (
    SELECT 'schema' AS source, se.table_name || '.' || se.column_name AS ref,
           se.description, 1 - (se.embedding <=> q.emb) AS similarity
    FROM schema_elements se, question_embedding q
    WHERE se.data_source_id = $2 AND se.is_visible = TRUE
    ORDER BY se.embedding <=> q.emb LIMIT 5
),
model_matches AS (
    SELECT 'model' AS source, sm.name AS ref,
           sm.description, 1 - (sm.embedding <=> q.emb) AS similarity
    FROM semantic_models sm, question_embedding q
    WHERE sm.organization_id = $3 AND sm.is_published = TRUE
    ORDER BY sm.embedding <=> q.emb LIMIT 3
),
glossary_matches AS (
    SELECT 'glossary' AS source, sg.term AS ref,
           sg.definition AS description, 1 - (sg.embedding <=> q.emb) AS similarity
    FROM semantic_glossary sg, question_embedding q
    WHERE sg.organization_id = $3
    ORDER BY sg.embedding <=> q.emb LIMIT 3
)
SELECT * FROM schema_matches
UNION ALL SELECT * FROM model_matches
UNION ALL SELECT * FROM glossary_matches
ORDER BY similarity DESC
LIMIT 10;
```

---

## 4. Conversations & Queries

### conversations

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           TEXT,
    data_source_id  UUID REFERENCES data_sources(id),  -- default data source for this conversation
    context         JSONB NOT NULL DEFAULT '{}',
    -- Conversation-level context accumulated across turns:
    -- {"active_filters": {"region": "US", "date_range": "2026-Q1"},
    --  "referenced_tables": ["orders", "customers"],
    --  "referenced_measures": ["total_revenue", "order_count"],
    --  "clarifications": [{"term": "revenue", "resolved_to": "total_revenue"}]}
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conversations_user ON conversations (user_id, updated_at DESC);
```

### messages

```sql
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content         TEXT NOT NULL,
    message_index   INTEGER NOT NULL,
    attachments     JSONB NOT NULL DEFAULT '[]',
    -- Example attachments (for assistant messages):
    -- [{"type": "query", "query_id": "uuid"},
    --  {"type": "visualization", "visualization_id": "uuid"},
    --  {"type": "insight", "text": "Revenue increased 23% QoQ..."}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_conversation ON messages (conversation_id, message_index);
```

### queries

```sql
CREATE TABLE queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),

    -- Input
    natural_language TEXT NOT NULL,

    -- Generation
    generation      JSONB NOT NULL,
    -- {"generated_sql": "SELECT region, SUM(total_amount) ...",
    --  "sql_dialect": "postgresql",
    --  "confidence_score": 0.92,
    --  "llm_model": "claude-sonnet-4-20250514",
    --  "llm_tokens_input": 2340,
    --  "llm_tokens_output": 156,
    --  "schema_context_used": ["orders.region", "orders.total_amount", "orders.created_at"],
    --  "semantic_models_used": ["orders"],
    --  "measures_used": ["total_revenue"],
    --  "dimensions_used": ["region", "order_date"],
    --  "decomposition_steps": [
    --    {"step": 1, "description": "Filter orders to Q1 2026", "sql_fragment": "WHERE ..."},
    --    {"step": 2, "description": "Group by region and sum revenue", "sql_fragment": "GROUP BY ..."}
    --  ]}

    -- Execution
    executed_sql    TEXT,                       -- final SQL after any dialect transformations
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                        'pending', 'generating', 'validating', 'executing',
                        'completed', 'error', 'cancelled'
                    )),
    execution       JSONB NOT NULL DEFAULT '{}',
    -- {"row_count": 5, "execution_time_ms": 234, "result_hash": "sha256:...",
    --  "error_message": null, "error_type": null,
    --  "corrections": [{"from_sql": "...", "to_sql": "...", "reason": "column_typo"}]}

    -- Result (inline for small results)
    result_columns  JSONB,                     -- [{"name": "region", "type": "text"}, ...]
    result_data     JSONB,                     -- [{"region": "US", "total_revenue": 1234567}, ...]
    result_truncated BOOLEAN NOT NULL DEFAULT FALSE,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_queries_conversation ON queries (conversation_id, created_at);
CREATE INDEX idx_queries_user ON queries (user_id, created_at DESC);
CREATE INDEX idx_queries_org ON queries (organization_id, created_at DESC);
CREATE INDEX idx_queries_status ON queries (organization_id, status) WHERE status NOT IN ('completed', 'cancelled');
CREATE INDEX idx_queries_generation ON queries USING GIN (generation);
```

---

## 5. Visualizations & Dashboards

### visualizations

```sql
CREATE TABLE visualizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID NOT NULL REFERENCES queries(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    title           TEXT,
    chart_type      TEXT NOT NULL,              -- 'bar', 'line', 'pie', 'table', etc.
    config          JSONB NOT NULL DEFAULT '{}',
    -- Config structure varies by chart_type:
    -- Bar: {"x_axis": "region", "y_axis": "total_revenue", "color_by": null,
    --        "orientation": "vertical", "sort": "desc", "limit": 20,
    --        "colors": {"palette": "default"}, "show_legend": true, "show_values": true}
    -- Line: {"x_axis": "order_date", "y_axis": ["total_revenue", "order_count"],
    --         "granularity": "month", "show_trend_line": true, "interpolation": "monotone"}
    -- Pie: {"dimension": "region", "measure": "total_revenue", "show_percentage": true,
    --        "max_slices": 8, "other_label": "Other"}
    -- Table: {"columns": ["region", "total_revenue", "order_count"],
    --          "sort_by": "total_revenue", "sort_dir": "desc", "page_size": 25,
    --          "conditional_formatting": [{"column": "total_revenue", "type": "heatmap"}]}
    insight_text    TEXT,                       -- LLM-generated narrative
    is_saved        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_viz_org ON visualizations (organization_id) WHERE is_saved = TRUE;
```

### dashboards

```sql
CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    description     TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    -- {"layout": [{"viz_id": "uuid", "x": 0, "y": 0, "w": 6, "h": 4}, ...],
    --  "filters": [{"dimension": "region", "type": "dropdown", "default": "all"},
    --              {"dimension": "date_range", "type": "date_picker", "default": "last_90_days"}],
    --  "refresh_interval_seconds": 300,
    --  "theme": {"background": "#ffffff", "grid_gap": 16}}
    sharing         JSONB NOT NULL DEFAULT '{}',
    -- {"users": [{"user_id": "uuid", "access": "edit"}],
    --  "roles": [{"role_id": "uuid", "access": "view"}],
    --  "public_link": {"token": "abc123", "expires_at": "2026-12-31T23:59:59Z"},
    --  "embed": {"token": "xyz789", "allowed_domains": ["*.company.com"]}}
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dashboards_org ON dashboards (organization_id);
CREATE INDEX idx_dashboards_sharing ON dashboards USING GIN (sharing);
```

---

## 6. Alerts

### alerts

```sql
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    config          JSONB NOT NULL,
    -- {"query_template": "SELECT SUM(amount) FROM orders WHERE created_at > now() - interval '1 day'",
    --  "condition": {"type": "threshold_above", "value": 10000},
    --  "schedule_cron": "0 9 * * *",
    --  "notifications": [
    --    {"channel": "slack", "webhook_url": "https://hooks.slack.com/..."},
    --    {"channel": "email", "recipients": ["analyst@company.com"]}
    --  ]}
    state           JSONB NOT NULL DEFAULT '{}',
    -- {"is_active": true, "last_evaluated_at": "2026-05-11T09:00:00Z",
    --  "last_triggered_at": "2026-05-10T09:00:00Z", "last_value": 12345,
    --  "trigger_count": 7, "consecutive_triggers": 2}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_org ON alerts (organization_id);
CREATE INDEX idx_alerts_active ON alerts (organization_id)
    WHERE (state->>'is_active')::boolean = TRUE;
```

---

## 7. Audit & LLM Configuration

### activity_log

```sql
CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details for a query action:
    -- {"natural_language": "What was revenue last quarter?",
    --  "confidence_score": 0.92, "execution_time_ms": 234,
    --  "tables_accessed": ["orders"], "row_count": 5}
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE activity_log_2026_q1 PARTITION OF activity_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE activity_log_2026_q2 PARTITION OF activity_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE activity_log_2026_q3 PARTITION OF activity_log
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE activity_log_2026_q4 PARTITION OF activity_log
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_activity_org ON activity_log (organization_id, created_at DESC);
CREATE INDEX idx_activity_user ON activity_log (user_id, created_at DESC) WHERE user_id IS NOT NULL;
```

### llm_config

```sql
CREATE TABLE llm_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    config_type     TEXT NOT NULL CHECK (config_type IN ('provider', 'prompt_template', 'embedding_model')),
    name            TEXT NOT NULL,
    config          JSONB NOT NULL,
    -- Provider example:
    -- {"provider_type": "anthropic", "api_endpoint": "https://api.anthropic.com",
    --  "api_key_encrypted": "...", "default_model": "claude-sonnet-4-20250514",
    --  "is_default": true}
    -- Prompt template example:
    -- {"purpose": "nl_to_sql", "version": 3, "is_active": true,
    --  "template": "You are a SQL expert. Given the following schema context:\n{schema_context}\n..."}
    -- Embedding model example:
    -- {"provider": "openai", "model": "text-embedding-3-small", "dimensions": 1536,
    --  "api_key_encrypted": "...", "is_default": true}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, config_type, name)
);

CREATE INDEX idx_llm_config_org ON llm_config (organization_id, config_type);
```

---

## 8. Schema Change Detection (Self-Healing)

### schema_change_log

```sql
-- Tracks changes detected during schema syncs for self-healing dashboard support
CREATE TABLE schema_change_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    change_type     TEXT NOT NULL CHECK (change_type IN (
                        'table_added', 'table_removed', 'table_renamed',
                        'column_added', 'column_removed', 'column_renamed',
                        'column_type_changed'
                    )),
    details         JSONB NOT NULL,
    -- Column renamed example:
    -- {"schema": "public", "table": "orders",
    --  "old_column": "total_amt", "new_column": "total_amount",
    --  "similarity_score": 0.94, "auto_mapped": true}
    resolution      JSONB,
    -- {"status": "auto_resolved", "affected_queries": 3, "affected_dashboards": 1,
    --  "mapping": {"total_amt": "total_amount"}}
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_schema_changes_source ON schema_change_log (data_source_id, detected_at DESC);
CREATE INDEX idx_schema_changes_unresolved ON schema_change_log (data_source_id)
    WHERE resolved_at IS NULL;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Access Control | 4 | organizations, users, roles, user_roles |
| Data Sources & Schema | 2 | data_sources, schema_elements (with vector embeddings) |
| Semantic Layer | 2 | semantic_models (with vector embeddings), semantic_glossary (with vector embeddings) |
| Conversations & Queries | 3 | conversations, messages, queries |
| Visualizations & Dashboards | 2 | visualizations, dashboards |
| Alerts | 1 | alerts (config + state in JSONB) |
| Audit & Config | 2 | activity_log (partitioned), llm_config |
| Schema Change Detection | 1 | schema_change_log |
| **Total** | **17** | Plus quarterly partitions for activity_log |

---

## Key Design Decisions

1. **Vector embeddings on schema elements, semantic models, and glossary terms.** This is the architectural centerpiece. Rather than sending 400 table/column descriptions to the LLM (which exceeds context windows and degrades accuracy), the system embeds each description and retrieves only the top-10 most relevant elements via cosine similarity. Research shows this RAG approach improves text-to-SQL accuracy by 15-25% on large schemas.

2. **Unified `schema_elements` table instead of separate tables/columns tables.** Tables and columns are stored in the same table with an `element_type` discriminator. This simplifies the vector search query (single table scan instead of UNION), reduces join complexity, and aligns with how the NL engine consumes schema context (as a flat list of descriptions, not a hierarchical tree).

3. **JSONB for connector-specific configuration.** Each database engine (PostgreSQL, Snowflake, BigQuery, Databricks) requires different connection parameters. Rather than a union of nullable columns or an EAV pattern, the `connection` JSONB column holds engine-specific structures validated at the application level. This eliminates migrations when adding new connector types.

4. **Semantic model definition stored as a single JSONB document.** Measures, dimensions, and joins are stored together in the `definition` JSONB column rather than in separate relational tables. This mirrors how Cube.js and dbt define models (as single YAML documents) and simplifies sync: write the entire model definition atomically rather than managing parent-child relationships across multiple tables.

5. **Business glossary with vector search for disambiguation.** The `semantic_glossary` table maps business jargon ("ARR", "MQL", "churn rate") to specific semantic model measures/dimensions. Vector similarity search on glossary entries helps the NL engine resolve ambiguous terms before SQL generation. This is a unique feature not present in the normalized model.

6. **Conversation context accumulates in JSONB.** The `conversations.context` JSONB field stores accumulated conversational state — active filters, referenced tables, resolved ambiguities — across turns. This enables multi-turn follow-ups ("now break that down by region") without re-parsing the entire conversation history.

7. **Schema change detection for self-healing dashboards.** The `schema_change_log` table tracks structural changes between schema syncs (renamed columns, added/removed tables). When a change is detected, the system uses embedding similarity to propose automatic mappings (e.g., `total_amt` -> `total_amount` with 94% similarity), enabling self-healing of affected queries and dashboards.

8. **Fewer tables, more JSONB, same expressiveness.** This model has 17 tables versus the normalized model's 27. The difference comes from consolidating junction tables, configuration tables, and child entity tables into JSONB fields on parent entities. The trade-off is that JSONB fields lack foreign key constraints, but the application layer provides validation via JSON Schema.
