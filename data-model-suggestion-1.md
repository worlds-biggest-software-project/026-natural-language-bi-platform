# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Natural Language BI Platform · Created: 2026-05-11

## Philosophy

This model follows a traditional normalized relational approach where every distinct concept in the system — data sources, semantic models, conversations, queries, visualizations, dashboards — gets its own table with well-defined foreign key relationships. The schema is designed for clarity, referential integrity, and straightforward querying by application developers.

The approach mirrors how mature BI platforms like Apache Superset and Metabase structure their internal application databases: a metadata layer describing connected databases, their schemas, and governed metrics, sitting alongside a content layer of user-created artifacts (saved questions, charts, dashboards). Every relationship is explicit and enforceable via foreign keys.

This model treats the semantic layer metadata (Cube.js cubes/measures/dimensions or dbt semantic models) as first-class relational entities within the application database, enabling the NL-to-SQL engine to query them directly rather than fetching them from external APIs at runtime.

**Best for:** Teams that prioritize data integrity, need complex cross-entity queries (e.g., "which dashboards reference metrics from this data source?"), and want a schema that is immediately understandable by any developer familiar with relational databases.

**Trade-offs:**
- (+) Maximum referential integrity — broken relationships are impossible at the database level
- (+) Standard SQL tooling works out of the box — no special query patterns needed
- (+) Easy to reason about for new developers joining the team
- (+) Well-suited for PostgreSQL row-level security for multi-tenant isolation
- (-) High table count (~35-40 tables) increases migration complexity
- (-) Schema changes require migrations for every new concept or attribute
- (-) Junction tables for many-to-many relationships add query complexity
- (-) Less flexible for jurisdiction-specific or connector-specific metadata that varies widely

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SQL:2023 | All DDL follows standard PostgreSQL syntax; no proprietary extensions except pgcrypto for UUIDs |
| Cube.js Data Model Spec | Semantic model tables mirror Cube.js cube/measure/dimension YAML structure for bidirectional sync |
| dbt Semantic Layer (MetricFlow) | Semantic entities/dimensions/measures map to MetricFlow's entity/dimension/metric types |
| ISO 8601 | All timestamps stored as TIMESTAMPTZ; durations in ISO 8601 interval format |
| OAuth 2.0 / OIDC | Auth tables support external identity providers with standard claims mapping |
| OpenTelemetry Semantic Conventions | Query execution logs follow OpenTelemetry attribute naming for observability integration |

---

## 1. Identity & Access Control

### organizations

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,       -- URL-safe identifier
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free', 'starter', 'pro', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}', -- org-wide config (default LLM provider, etc.)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_slug ON organizations (slug);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL DEFAULT 'local' CHECK (auth_provider IN ('local', 'google', 'github', 'saml', 'oidc')),
    auth_subject    TEXT,                       -- external IdP subject identifier
    password_hash   TEXT,                       -- NULL for SSO-only users
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, email)
);

CREATE INDEX idx_users_org ON users (organization_id);
CREATE INDEX idx_users_email ON users (email);
```

### roles

```sql
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,              -- 'admin', 'analyst', 'viewer', 'editor'
    description     TEXT,
    permissions     TEXT[] NOT NULL DEFAULT '{}', -- array of permission strings
    is_system       BOOLEAN NOT NULL DEFAULT FALSE, -- TRUE for built-in roles
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);
```

### user_roles

```sql
CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by      UUID REFERENCES users(id) ON DELETE SET NULL,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

### api_keys

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL,              -- bcrypt hash of the API key
    key_prefix      TEXT NOT NULL,              -- first 8 chars for identification
    scopes          TEXT[] NOT NULL DEFAULT '{}', -- 'read:queries', 'write:dashboards', etc.
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_prefix ON api_keys (key_prefix) WHERE is_active = TRUE;
```

---

## 2. Data Source Management

### data_sources

```sql
CREATE TABLE data_sources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    engine          TEXT NOT NULL CHECK (engine IN (
                        'postgresql', 'mysql', 'bigquery', 'snowflake',
                        'redshift', 'databricks', 'clickhouse', 'trino', 'duckdb'
                    )),
    connection_config JSONB NOT NULL,           -- encrypted connection parameters
    -- Example connection_config for PostgreSQL:
    -- {"host": "db.example.com", "port": 5432, "database": "analytics",
    --  "username": "readonly", "ssl_mode": "require"}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_synced_at  TIMESTAMPTZ,               -- last schema sync timestamp
    sync_status     TEXT NOT NULL DEFAULT 'pending' CHECK (sync_status IN ('pending', 'syncing', 'synced', 'error')),
    sync_error      TEXT,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources (organization_id);
```

### schema_tables

```sql
CREATE TABLE schema_tables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    schema_name     TEXT NOT NULL,              -- e.g., 'public', 'analytics'
    table_name      TEXT NOT NULL,
    table_type      TEXT NOT NULL DEFAULT 'table' CHECK (table_type IN ('table', 'view', 'materialized_view')),
    description     TEXT,                       -- human-authored or LLM-generated description
    row_count_estimate BIGINT,                  -- from pg_stat or equivalent
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE, -- hide from NL query engine
    last_synced_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_source_id, schema_name, table_name)
);

CREATE INDEX idx_schema_tables_source ON schema_tables (data_source_id);
```

### schema_columns

```sql
CREATE TABLE schema_columns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES schema_tables(id) ON DELETE CASCADE,
    column_name     TEXT NOT NULL,
    data_type       TEXT NOT NULL,              -- native type string from the source DB
    is_nullable     BOOLEAN NOT NULL DEFAULT TRUE,
    is_primary_key  BOOLEAN NOT NULL DEFAULT FALSE,
    is_foreign_key  BOOLEAN NOT NULL DEFAULT FALSE,
    foreign_table   TEXT,                       -- referenced table (schema.table)
    foreign_column  TEXT,                       -- referenced column
    description     TEXT,                       -- human-authored or LLM-generated
    sample_values   TEXT[],                     -- up to 10 sample values for LLM context
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,
    ordinal_position INTEGER NOT NULL,
    UNIQUE (table_id, column_name)
);

CREATE INDEX idx_schema_columns_table ON schema_columns (table_id);
```

### data_source_permissions

```sql
CREATE TABLE data_source_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    access_level    TEXT NOT NULL CHECK (access_level IN ('none', 'read', 'admin')),
    table_filter    TEXT[],                     -- NULL = all tables; specific table names to restrict
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_source_id, role_id)
);
```

---

## 3. Semantic Layer

### semantic_models

```sql
CREATE TABLE semantic_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    data_source_id  UUID NOT NULL REFERENCES data_sources(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,              -- e.g., 'orders', 'customers'
    display_name    TEXT NOT NULL,
    description     TEXT,
    source_type     TEXT NOT NULL CHECK (source_type IN ('cubejs', 'dbt', 'custom')),
    sql_table       TEXT,                       -- base table or SQL expression
    external_ref    TEXT,                       -- Cube.js cube name or dbt model ref
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);

CREATE INDEX idx_semantic_models_org ON semantic_models (organization_id);
CREATE INDEX idx_semantic_models_source ON semantic_models (data_source_id);
```

### semantic_measures

```sql
CREATE TABLE semantic_measures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    semantic_model_id UUID NOT NULL REFERENCES semantic_models(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,              -- e.g., 'total_revenue', 'order_count'
    display_name    TEXT NOT NULL,
    description     TEXT,                       -- business definition for LLM grounding
    sql_expression  TEXT NOT NULL,              -- e.g., 'SUM(amount)', 'COUNT(DISTINCT user_id)'
    agg_type        TEXT NOT NULL CHECK (agg_type IN (
                        'count', 'count_distinct', 'sum', 'avg', 'min', 'max',
                        'number', 'ratio', 'cumulative', 'derived'
                    )),
    data_type       TEXT NOT NULL DEFAULT 'number' CHECK (data_type IN ('number', 'currency', 'percentage')),
    format_string   TEXT,                       -- e.g., '$#,##0.00'
    filters         JSONB,                      -- default filters applied to this measure
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,
    ordinal_position INTEGER NOT NULL DEFAULT 0,
    UNIQUE (semantic_model_id, name)
);

CREATE INDEX idx_semantic_measures_model ON semantic_measures (semantic_model_id);
```

### semantic_dimensions

```sql
CREATE TABLE semantic_dimensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    semantic_model_id UUID NOT NULL REFERENCES semantic_models(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,              -- e.g., 'order_date', 'customer_region'
    display_name    TEXT NOT NULL,
    description     TEXT,                       -- business definition for LLM grounding
    sql_expression  TEXT NOT NULL,              -- column reference or expression
    dim_type        TEXT NOT NULL CHECK (dim_type IN (
                        'string', 'number', 'time', 'boolean', 'geo'
                    )),
    time_granularity TEXT CHECK (                -- only for time dimensions
                        time_granularity IN ('second', 'minute', 'hour', 'day', 'week', 'month', 'quarter', 'year')
                        OR time_granularity IS NULL
                    ),
    is_primary_key  BOOLEAN NOT NULL DEFAULT FALSE,
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,
    ordinal_position INTEGER NOT NULL DEFAULT 0,
    UNIQUE (semantic_model_id, name)
);

CREATE INDEX idx_semantic_dimensions_model ON semantic_dimensions (semantic_model_id);
```

### semantic_joins

```sql
CREATE TABLE semantic_joins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    from_model_id   UUID NOT NULL REFERENCES semantic_models(id) ON DELETE CASCADE,
    to_model_id     UUID NOT NULL REFERENCES semantic_models(id) ON DELETE CASCADE,
    join_type       TEXT NOT NULL CHECK (join_type IN ('inner', 'left', 'right', 'full', 'cross')),
    sql_on          TEXT NOT NULL,              -- e.g., '{from}.user_id = {to}.id'
    relationship    TEXT NOT NULL CHECK (relationship IN ('one_to_one', 'one_to_many', 'many_to_one', 'many_to_many')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (from_model_id, to_model_id)
);
```

---

## 4. Conversations & Queries

### conversations

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title           TEXT,                       -- auto-generated or user-provided
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_conversations_user ON conversations (user_id, created_at DESC);
CREATE INDEX idx_conversations_org ON conversations (organization_id, created_at DESC);
```

### messages

```sql
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content         TEXT NOT NULL,              -- natural language text
    message_index   INTEGER NOT NULL,           -- ordering within conversation
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
    natural_language TEXT NOT NULL,             -- original user question
    generated_sql   TEXT NOT NULL,              -- LLM-generated SQL
    executed_sql    TEXT,                        -- final SQL after any transformations
    sql_dialect     TEXT NOT NULL,              -- 'postgresql', 'snowflake', etc.
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                        'pending', 'generating', 'validating', 'executing',
                        'completed', 'error', 'cancelled'
                    )),
    confidence_score NUMERIC(4,3),             -- 0.000 to 1.000
    error_message   TEXT,
    row_count       INTEGER,
    execution_time_ms INTEGER,
    llm_model       TEXT,                       -- e.g., 'claude-sonnet-4-20250514'
    llm_tokens_used INTEGER,
    result_hash     TEXT,                       -- SHA-256 of result for caching
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_queries_conversation ON queries (conversation_id, created_at);
CREATE INDEX idx_queries_user ON queries (user_id, created_at DESC);
CREATE INDEX idx_queries_org_status ON queries (organization_id, status);
CREATE INDEX idx_queries_result_hash ON queries (result_hash) WHERE result_hash IS NOT NULL;
```

### query_results

```sql
CREATE TABLE query_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID NOT NULL REFERENCES queries(id) ON DELETE CASCADE,
    columns         JSONB NOT NULL,             -- [{name, type, display_name}]
    data            JSONB NOT NULL,             -- array of row objects (capped at 10K rows)
    -- Example:
    -- columns: [{"name": "region", "type": "string"}, {"name": "revenue", "type": "number"}]
    -- data: [{"region": "US", "revenue": 1234567}, {"region": "EU", "revenue": 987654}]
    truncated       BOOLEAN NOT NULL DEFAULT FALSE,
    total_rows      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_query_results_query ON query_results (query_id);
```

### query_semantic_refs

```sql
-- Tracks which semantic models/measures/dimensions a query referenced
CREATE TABLE query_semantic_refs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID NOT NULL REFERENCES queries(id) ON DELETE CASCADE,
    semantic_model_id UUID REFERENCES semantic_models(id) ON DELETE SET NULL,
    semantic_measure_id UUID REFERENCES semantic_measures(id) ON DELETE SET NULL,
    semantic_dimension_id UUID REFERENCES semantic_dimensions(id) ON DELETE SET NULL
);

CREATE INDEX idx_query_semantic_refs_query ON query_semantic_refs (query_id);
CREATE INDEX idx_query_semantic_refs_model ON query_semantic_refs (semantic_model_id);
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
    chart_type      TEXT NOT NULL CHECK (chart_type IN (
                        'table', 'bar', 'line', 'area', 'pie', 'donut', 'scatter',
                        'heatmap', 'treemap', 'funnel', 'gauge', 'map',
                        'stacked_bar', 'grouped_bar', 'combo', 'histogram',
                        'waterfall', 'bubble', 'radar', 'number'
                    )),
    config          JSONB NOT NULL DEFAULT '{}', -- chart configuration (axes, colors, labels)
    -- Example config:
    -- {"x_axis": "region", "y_axis": "revenue", "color_by": "product_category",
    --  "sort": "desc", "limit": 20, "show_legend": true}
    insight_text    TEXT,                       -- LLM-generated narrative about the data
    is_saved        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_visualizations_user ON visualizations (user_id, created_at DESC);
CREATE INDEX idx_visualizations_org ON visualizations (organization_id) WHERE is_saved = TRUE;
```

### dashboards

```sql
CREATE TABLE dashboards (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    title           TEXT NOT NULL,
    description     TEXT,
    is_published    BOOLEAN NOT NULL DEFAULT FALSE,
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE,
    layout          JSONB NOT NULL DEFAULT '[]', -- grid layout positions
    -- Example layout:
    -- [{"viz_id": "...", "x": 0, "y": 0, "w": 6, "h": 4},
    --  {"viz_id": "...", "x": 6, "y": 0, "w": 6, "h": 4}]
    filters         JSONB NOT NULL DEFAULT '[]', -- dashboard-level filters
    refresh_interval_seconds INTEGER,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dashboards_org ON dashboards (organization_id);
CREATE INDEX idx_dashboards_creator ON dashboards (created_by);
```

### dashboard_visualizations

```sql
CREATE TABLE dashboard_visualizations (
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    visualization_id UUID NOT NULL REFERENCES visualizations(id) ON DELETE CASCADE,
    position_config JSONB NOT NULL DEFAULT '{}', -- x, y, w, h for grid layout
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (dashboard_id, visualization_id)
);
```

### dashboard_shares

```sql
CREATE TABLE dashboard_shares (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dashboard_id    UUID NOT NULL REFERENCES dashboards(id) ON DELETE CASCADE,
    share_type      TEXT NOT NULL CHECK (share_type IN ('user', 'role', 'link', 'embed')),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID REFERENCES roles(id) ON DELETE CASCADE,
    share_token     TEXT UNIQUE,                -- for link/embed sharing
    access_level    TEXT NOT NULL CHECK (access_level IN ('view', 'edit')),
    expires_at      TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dashboard_shares_dashboard ON dashboard_shares (dashboard_id);
CREATE INDEX idx_dashboard_shares_token ON dashboard_shares (share_token) WHERE share_token IS NOT NULL;
```

---

## 6. Alerts & Scheduled Reports

### alerts

```sql
CREATE TABLE alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    query_template  TEXT NOT NULL,              -- parameterized SQL or NL query
    data_source_id  UUID NOT NULL REFERENCES data_sources(id),
    condition_type  TEXT NOT NULL CHECK (condition_type IN (
                        'threshold_above', 'threshold_below', 'change_percent',
                        'anomaly', 'no_data'
                    )),
    condition_value NUMERIC,
    schedule_cron   TEXT NOT NULL,              -- cron expression
    notification_channels JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"type": "slack", "webhook_url": "..."}, {"type": "email", "to": ["..."]}]
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_evaluated_at TIMESTAMPTZ,
    last_triggered_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alerts_org ON alerts (organization_id);
CREATE INDEX idx_alerts_schedule ON alerts (is_active, schedule_cron);
```

### alert_history

```sql
CREATE TABLE alert_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id        UUID NOT NULL REFERENCES alerts(id) ON DELETE CASCADE,
    evaluated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    result_value    NUMERIC,
    triggered       BOOLEAN NOT NULL,
    notification_sent BOOLEAN NOT NULL DEFAULT FALSE,
    error_message   TEXT
);

CREATE INDEX idx_alert_history_alert ON alert_history (alert_id, evaluated_at DESC);
```

---

## 7. Audit & Activity Logging

### activity_log

```sql
CREATE TABLE activity_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL,              -- 'query.created', 'dashboard.shared', etc.
    resource_type   TEXT NOT NULL,              -- 'query', 'dashboard', 'data_source', etc.
    resource_id     UUID,
    metadata        JSONB NOT NULL DEFAULT '{}', -- action-specific details
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_activity_log_org ON activity_log (organization_id, created_at DESC);
CREATE INDEX idx_activity_log_user ON activity_log (user_id, created_at DESC);
CREATE INDEX idx_activity_log_resource ON activity_log (resource_type, resource_id);
```

---

## 8. LLM Configuration

### llm_providers

```sql
CREATE TABLE llm_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,              -- 'anthropic', 'openai', 'azure_openai'
    provider_type   TEXT NOT NULL CHECK (provider_type IN ('anthropic', 'openai', 'azure_openai', 'custom')),
    api_endpoint    TEXT,
    api_key_encrypted TEXT NOT NULL,            -- encrypted API key
    default_model   TEXT NOT NULL,              -- e.g., 'claude-sonnet-4-20250514'
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name)
);
```

### prompt_templates

```sql
CREATE TABLE prompt_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    purpose         TEXT NOT NULL CHECK (purpose IN (
                        'nl_to_sql', 'insight_narration', 'query_decomposition',
                        'schema_description', 'error_correction', 'chart_suggestion'
                    )),
    template_text   TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, name, version)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Access Control | 5 | organizations, users, roles, user_roles, api_keys |
| Data Source Management | 4 | data_sources, schema_tables, schema_columns, data_source_permissions |
| Semantic Layer | 4 | semantic_models, semantic_measures, semantic_dimensions, semantic_joins |
| Conversations & Queries | 5 | conversations, messages, queries, query_results, query_semantic_refs |
| Visualizations & Dashboards | 4 | visualizations, dashboards, dashboard_visualizations, dashboard_shares |
| Alerts & Scheduled Reports | 2 | alerts, alert_history |
| Audit & Activity | 1 | activity_log |
| LLM Configuration | 2 | llm_providers, prompt_templates |
| **Total** | **27** | |

---

## Key Design Decisions

1. **Semantic layer stored relationally, not just cached.** Measures, dimensions, and joins from Cube.js or dbt are materialized into the application database so the NL-to-SQL engine can query them via standard SQL joins rather than API calls at generation time. This eliminates a network round-trip from the critical path.

2. **Conversations and messages are separate from queries.** A conversation contains multiple messages (user and assistant turns), and each assistant response may generate zero or more SQL queries. This separation supports multi-turn context: the LLM receives the full message history to understand follow-up questions like "now break that down by region."

3. **Query results stored as JSONB, not in a separate data warehouse.** Results are capped at 10K rows and stored alongside the query record. This simplifies the architecture (no external result cache) at the cost of increased database size. A `result_hash` column enables deduplication and caching of identical queries.

4. **Row-Level Security via organization_id on every content table.** Every user-generated artifact (queries, dashboards, conversations) carries an `organization_id` foreign key, enabling PostgreSQL RLS policies to enforce tenant isolation without application-level filtering.

5. **Schema metadata synced from source databases, not queried live.** The `schema_tables` and `schema_columns` tables store a snapshot of each connected database's schema. This allows the NL engine to include table descriptions and sample values in its context without connecting to the source database during query generation.

6. **Visualizations are query-bound, not standalone.** Every visualization references a specific query result. This creates a clear provenance chain: natural language question -> SQL query -> result set -> visualization -> dashboard. Any visualization can be traced back to its source question.

7. **LLM provider configuration is per-organization.** Each organization can bring their own API keys and choose their preferred model. Prompt templates are versioned and organization-scoped, enabling A/B testing of different prompt strategies.

8. **Audit logging is append-only with no foreign key constraints on resource_id.** The activity_log references resource IDs but does not enforce foreign keys, so deleting a dashboard does not cascade-delete its audit history. This preserves the audit trail even after resources are removed.
