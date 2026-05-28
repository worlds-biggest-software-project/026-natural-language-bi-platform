# Natural Language BI Platform — Development Plan

> Project: Candidate #026 · Created: 2026-05-25

---

## Technology Decisions

These decisions are derived from the research survey (11 competing products analysed), the feature specification (table-stakes, differentiators, and underserved areas), and the three data model proposals. Each decision includes the rationale grounded in the research.

### Data Model: Hybrid Relational + JSONB with Vector Embeddings (Suggestion 3)

**Chosen over** Suggestion 1 (pure normalized) and Suggestion 2 (event-sourced CQRS).

**Rationale:**
- Vector embeddings on schema elements and semantic models are the single highest-impact architectural decision for NL-to-SQL accuracy. Research shows RAG-based schema retrieval improves text-to-SQL accuracy by 15-25% on large schemas (features.md, DIN-SQL / DAIL-SQL approaches). Without embeddings, sending full schemas to the LLM exceeds context windows and degrades accuracy — the exact failure mode that blocks adoption of current open-source tools.
- JSONB flexibility eliminates migrations when adding new database connectors (each engine has different connection parameters) and new chart types (each has different config shapes). This accelerates MVP delivery.
- 17 tables vs 27 (normalized) or 20+ (CQRS) reduces migration complexity and cognitive load for early contributors.
- Event sourcing (Suggestion 2) adds audit-first temporal querying but introduces projection infrastructure, eventual consistency, and a steeper learning curve. This capability can be layered in later (Phase 10) for regulated-industry deployments. The activity_log table in Suggestion 3 provides adequate auditing for MVP and v1.1.
- pgvector is widely available on managed PostgreSQL services (Supabase, Neon, RDS, Cloud SQL, Azure), keeping the "single database" deployment promise.

### Language & Runtime: TypeScript (Node.js)

**Rationale:**
- Cube.js SDK, the chosen semantic layer, is TypeScript-native. The Cube.js AI API and REST/GraphQL clients are first-class TypeScript consumers.
- The frontend (React/Next.js) shares the language, enabling full-stack type safety with shared types between API and UI.
- The open-source BI ecosystem (Metabase uses Clojure, Superset uses Python) lacks a TypeScript-native NL BI platform — this is a differentiator for contributor onboarding since TypeScript is the most widely-used language on GitHub.
- Anthropic and OpenAI SDKs have mature TypeScript clients.

### Framework: Next.js 15 (App Router) + tRPC

**Rationale:**
- Next.js provides SSR for initial dashboard loads (SEO for public dashboards, fast first paint) and API routes for the query engine.
- tRPC provides end-to-end type safety between the React frontend and the query/conversation APIs without code generation.
- App Router's React Server Components reduce client bundle size for dashboard views.
- Vercel deployment is straightforward for the managed SaaS tier; self-hosting via Docker is well-documented for Next.js.

### Database: PostgreSQL 16+ with pgvector

**Rationale:**
- The data model requires vector similarity search (schema RAG), JSONB indexing (GIN), and row-level security (multi-tenancy). PostgreSQL is the only database that provides all three in a single engine.
- pgvector's HNSW index algorithm provides fast approximate nearest neighbor search suitable for the schema retrieval workload (hundreds to low-thousands of vectors per data source, not millions).
- PostgreSQL RLS policies on `organization_id` enforce tenant isolation at the database level, the strongest guarantee available.

### Semantic Layer: Cube.js (primary), dbt Semantic Layer (secondary)

**Rationale:**
- Cube.js is the only open-source semantic layer with a production-ready AI API (features.md). Its Apache 2.0 licence has no copyleft or patent encumbrances.
- Cube.js REST, GraphQL, and SQL APIs provide multiple integration paths. The AI API enables direct LLM-to-semantic-layer communication.
- dbt Semantic Layer (MetricFlow) support is secondary — many target customers already use dbt, and supporting both widens the addressable market.
- The semantic layer is the accuracy anchor: research shows NL-to-SQL accuracy improves from 70-85% baseline to 90%+ when LLM output is constrained to governed metrics (README.md).

### LLM Integration: Anthropic Claude (primary), OpenAI (secondary), pluggable

**Rationale:**
- Claude's large context window (200K tokens) accommodates schema context, conversation history, and few-shot examples in a single prompt.
- Multi-provider support is essential: enterprises have existing LLM contracts and preferences. The LLM provider is configured per-organization in the data model.
- Embedding model: OpenAI text-embedding-3-small (1536 dimensions) for schema/glossary embeddings due to cost efficiency and broad adoption. Voyage-3 (1024 dimensions) as an alternative.

### Visualization: Custom React chart components (ECharts/Recharts)

**Rationale:**
- Embedding Apache Superset as a UI layer (considered in README.md) adds significant operational complexity (Python backend, Celery workers, Redis) and tight coupling.
- Building custom chart components on ECharts or Recharts provides full control over the NL-to-visualization pipeline and enables the embeddable widget use case (features.md: "embeddable NL query widget with signed-URL embedding API").
- Starting with 8-10 chart types (table, bar, line, area, pie, scatter, number/KPI, stacked bar) covers 90%+ of NL query result visualizations.

### Authentication: NextAuth.js (Auth.js v5)

**Rationale:**
- Supports local credentials, Google, GitHub, and SAML/OIDC — all auth providers specified in the data model.
- Integrates natively with Next.js App Router.
- Open-source (ISC licence) with no vendor lock-in.

### Deployment: Docker Compose (self-hosted), Vercel + Neon/Supabase (managed SaaS)

**Rationale:**
- Docker Compose is the standard self-hosted deployment for open-source BI tools (Metabase, Superset, Lightdash all use it).
- Vercel + managed PostgreSQL (Neon or Supabase, both support pgvector) for the managed SaaS tier.
- Kubernetes Helm chart deferred to Phase 9 — Docker Compose is sufficient for initial adoption.

---

## Project Structure

```
nlbi/
├── apps/
│   └── web/                          # Next.js 15 application
│       ├── src/
│       │   ├── app/                   # App Router pages & layouts
│       │   │   ├── (auth)/            # Login, register, SSO callback
│       │   │   ├── (dashboard)/       # Main application shell
│       │   │   │   ├── ask/           # NL query interface (conversation view)
│       │   │   │   ├── dashboards/    # Dashboard list, editor, viewer
│       │   │   │   ├── data-sources/  # Connection management & schema browser
│       │   │   │   ├── semantic/      # Semantic model editor & glossary
│       │   │   │   ├── alerts/        # Alert configuration & history
│       │   │   │   └── settings/      # Org settings, users, roles, LLM config
│       │   │   ├── api/               # API routes (tRPC handler, webhooks)
│       │   │   └── embed/             # Embeddable widget routes
│       │   ├── components/
│       │   │   ├── charts/            # ECharts/Recharts wrappers per chart type
│       │   │   ├── conversation/      # Chat UI, message bubbles, query cards
│       │   │   ├── dashboard/         # Grid layout, filter bar, widget frames
│       │   │   ├── schema/            # Schema browser tree, column inspector
│       │   │   └── ui/               # Shared design system (shadcn/ui base)
│       │   ├── lib/
│       │   │   ├── auth/              # NextAuth config, session helpers
│       │   │   ├── db/                # Drizzle ORM schema, migrations, client
│       │   │   ├── trpc/              # tRPC router definitions
│       │   │   └── utils/             # Shared utilities
│       │   └── styles/
│       ├── drizzle/                   # Migration files
│       └── public/
├── packages/
│   ├── query-engine/                  # NL-to-SQL pipeline (core differentiator)
│   │   ├── src/
│   │   │   ├── pipeline/              # Orchestrates the full NL-to-SQL flow
│   │   │   │   ├── context-assembler.ts   # RAG: embed question, retrieve schema context
│   │   │   │   ├── query-generator.ts     # LLM prompt construction & SQL generation
│   │   │   │   ├── query-validator.ts     # SQL parsing, safety checks, dialect validation
│   │   │   │   ├── query-executor.ts      # Execute against source DB, stream results
│   │   │   │   ├── query-corrector.ts     # Auto-correct failed queries (retry with error context)
│   │   │   │   └── decomposer.ts          # Multi-step query decomposition (DIN-SQL approach)
│   │   │   ├── semantic/              # Semantic layer integration
│   │   │   │   ├── cube-adapter.ts    # Cube.js REST/GraphQL client
│   │   │   │   ├── dbt-adapter.ts     # dbt Semantic Layer (MetricFlow) client
│   │   │   │   └── sync.ts           # Schema sync & embedding generation
│   │   │   ├── connectors/            # Database connector abstractions
│   │   │   │   ├── base.ts
│   │   │   │   ├── postgresql.ts
│   │   │   │   ├── mysql.ts
│   │   │   │   ├── snowflake.ts
│   │   │   │   └── bigquery.ts
│   │   │   ├── llm/                   # LLM provider abstraction
│   │   │   │   ├── provider.ts        # Interface + factory
│   │   │   │   ├── anthropic.ts       # Claude adapter
│   │   │   │   └── openai.ts          # OpenAI adapter
│   │   │   └── embeddings/            # Embedding generation & retrieval
│   │   │       ├── embedder.ts        # Generate embeddings for schema elements
│   │   │       └── retriever.ts       # Vector similarity search over pgvector
│   │   └── tests/
│   ├── chart-engine/                  # Visualization logic
│   │   ├── src/
│   │   │   ├── recommender.ts         # Suggest chart type from query result shape
│   │   │   ├── renderer.ts            # Chart config generation
│   │   │   └── types.ts
│   │   └── tests/
│   ├── insight-engine/                # LLM-generated narratives (Phase 7)
│   │   └── src/
│   └── embed-sdk/                     # Embeddable widget SDK (Phase 8)
│       └── src/
├── docker/
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── docker-compose.dev.yml
├── docs/
│   ├── architecture.md
│   ├── self-hosting.md
│   └── api-reference.md
├── turbo.json                         # Turborepo config
├── package.json
└── tsconfig.json
```

---

## Phased Development Plan

### Phase 1: Foundation — Database, Auth, and Project Skeleton

**Goal:** A deployable Next.js application with database migrations, authentication, and the organizational multi-tenancy foundation. No NL features yet — this is the skeleton that everything builds on.

**What:**
- Initialize monorepo with Turborepo, pnpm workspaces
- Set up Next.js 15 App Router with tRPC
- Define Drizzle ORM schema for identity & access control tables: `organizations`, `users`, `roles`, `user_roles`
- Install and configure pgvector extension in the PostgreSQL setup
- Implement NextAuth.js with local credentials and Google OAuth
- Build organization creation, user invitation, and role management flows
- Create the application shell layout (sidebar, topbar, routing)
- Set up Docker Compose for local development (PostgreSQL 16 + pgvector, Next.js dev server)

**Design:**

Drizzle schema for the core identity tables:

```typescript
// packages/query-engine is not yet involved — this is apps/web/src/lib/db/schema/

import { pgTable, uuid, text, boolean, timestamp, jsonb } from 'drizzle-orm/pg-core';
import { vector } from 'drizzle-orm/pg-core'; // pgvector support via drizzle-orm

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  plan: text('plan').notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  email: text('email').notNull(),
  displayName: text('display_name').notNull(),
  avatarUrl: text('avatar_url'),
  auth: jsonb('auth').notNull().default({}),
  roles: text('roles').array().notNull().default(['{viewer}']),
  preferences: jsonb('preferences').notNull().default({}),
  isActive: boolean('is_active').notNull().default(true),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Docker Compose for development:

```yaml
# docker/docker-compose.dev.yml
services:
  db:
    image: pgvector/pgvector:pg16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: nlbi_dev
      POSTGRES_USER: nlbi
      POSTGRES_PASSWORD: dev_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nlbi"]
      interval: 5s
      timeout: 5s
      retries: 5

  web:
    build:
      context: ..
      dockerfile: docker/Dockerfile
      target: development
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://nlbi:dev_password@db:5432/nlbi_dev
      NEXTAUTH_SECRET: dev-secret-change-in-production
      NEXTAUTH_URL: http://localhost:3000
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ../apps/web/src:/app/apps/web/src
      - ../packages:/app/packages

volumes:
  pgdata:
```

**Testing:**
- Database migrations apply cleanly on a fresh PostgreSQL 16 + pgvector instance
- User can register with email/password, log in, and see the application shell
- Google OAuth flow completes and creates a user record linked to an organization
- Organization slug uniqueness is enforced at the database level
- RLS policy on `organization_id` prevents cross-tenant data access (unit test with two orgs)
- Docker Compose `up` starts the full stack from scratch in under 60 seconds

**Estimated duration:** 2 weeks

---

### Phase 2: Data Source Connectors and Schema Sync

**Goal:** Users can connect PostgreSQL and MySQL databases, the system introspects the schema (tables, columns, types, foreign keys), stores the metadata, and generates vector embeddings for every schema element.

**What:**
- Define Drizzle schema for `data_sources` and `schema_elements` tables (with pgvector column)
- Build the connector abstraction layer (`packages/query-engine/src/connectors/`)
- Implement PostgreSQL and MySQL connectors (schema introspection via `information_schema`)
- Build the schema sync pipeline: connect -> introspect -> store metadata -> generate embeddings
- Build the embedding pipeline: for each schema element, construct a description string, call the embedding API, store the vector
- Create the data source management UI: add connection, test connection, trigger sync, browse schema tree
- Implement connection credential encryption (AES-256, key from environment variable)

**Design:**

Connector interface and PostgreSQL implementation:

```typescript
// packages/query-engine/src/connectors/base.ts

export interface SchemaElement {
  elementType: 'table' | 'column';
  schemaName: string;
  tableName: string;
  columnName?: string;
  dataType?: string;
  description?: string;
  metadata: Record<string, unknown>;
}

export interface QueryResult {
  columns: Array<{ name: string; type: string }>;
  rows: Record<string, unknown>[];
  rowCount: number;
  executionTimeMs: number;
}

export interface DatabaseConnector {
  testConnection(): Promise<{ ok: boolean; error?: string }>;
  introspectSchema(): Promise<SchemaElement[]>;
  executeQuery(sql: string, params?: unknown[], timeoutMs?: number): Promise<QueryResult>;
  getDialect(): string;
  close(): Promise<void>;
}

// packages/query-engine/src/connectors/postgresql.ts

import { Pool, type PoolConfig } from 'pg';
import type { DatabaseConnector, SchemaElement, QueryResult } from './base';

export class PostgreSQLConnector implements DatabaseConnector {
  private pool: Pool;

  constructor(config: PoolConfig) {
    this.pool = new Pool({ ...config, max: 5, idleTimeoutMillis: 30000 });
  }

  async introspectSchema(): Promise<SchemaElement[]> {
    const elements: SchemaElement[] = [];

    // Fetch tables
    const tables = await this.pool.query(`
      SELECT
        t.table_schema,
        t.table_name,
        t.table_type,
        pg_stat.n_live_tup AS row_count_estimate,
        obj_description((t.table_schema || '.' || t.table_name)::regclass) AS table_comment
      FROM information_schema.tables t
      LEFT JOIN pg_stat_user_tables pg_stat
        ON pg_stat.schemaname = t.table_schema AND pg_stat.relname = t.table_name
      WHERE t.table_schema NOT IN ('pg_catalog', 'information_schema')
      ORDER BY t.table_schema, t.table_name
    `);

    for (const table of tables.rows) {
      elements.push({
        elementType: 'table',
        schemaName: table.table_schema,
        tableName: table.table_name,
        description: table.table_comment,
        metadata: {
          tableType: table.table_type === 'BASE TABLE' ? 'table' : 'view',
          rowCountEstimate: Number(table.row_count_estimate) || null,
        },
      });
    }

    // Fetch columns with foreign key info
    const columns = await this.pool.query(`
      SELECT
        c.table_schema,
        c.table_name,
        c.column_name,
        c.data_type,
        c.is_nullable,
        c.ordinal_position,
        col_description((c.table_schema || '.' || c.table_name)::regclass, c.ordinal_position) AS column_comment,
        tc.constraint_type,
        ccu.table_name AS foreign_table,
        ccu.column_name AS foreign_column
      FROM information_schema.columns c
      LEFT JOIN information_schema.key_column_usage kcu
        ON c.table_schema = kcu.table_schema AND c.table_name = kcu.table_name AND c.column_name = kcu.column_name
      LEFT JOIN information_schema.table_constraints tc
        ON kcu.constraint_name = tc.constraint_name AND kcu.table_schema = tc.table_schema
      LEFT JOIN information_schema.constraint_column_usage ccu
        ON tc.constraint_name = ccu.constraint_name AND tc.table_schema = ccu.table_schema
        AND tc.constraint_type = 'FOREIGN KEY'
      WHERE c.table_schema NOT IN ('pg_catalog', 'information_schema')
      ORDER BY c.table_schema, c.table_name, c.ordinal_position
    `);

    for (const col of columns.rows) {
      elements.push({
        elementType: 'column',
        schemaName: col.table_schema,
        tableName: col.table_name,
        columnName: col.column_name,
        dataType: col.data_type,
        description: col.column_comment,
        metadata: {
          isNullable: col.is_nullable === 'YES',
          isPrimaryKey: col.constraint_type === 'PRIMARY KEY',
          isForeignKey: col.constraint_type === 'FOREIGN KEY',
          foreignTable: col.foreign_table,
          foreignColumn: col.foreign_column,
          ordinalPosition: col.ordinal_position,
        },
      });
    }

    return elements;
  }

  getDialect(): string {
    return 'postgresql';
  }

  // ... testConnection(), executeQuery(), close() implementations
}
```

Embedding pipeline:

```typescript
// packages/query-engine/src/embeddings/embedder.ts

import type { SchemaElement } from '../connectors/base';

export interface EmbeddingProvider {
  embed(texts: string[]): Promise<number[][]>;
  dimensions(): number;
}

export function buildEmbeddingText(element: SchemaElement): string {
  if (element.elementType === 'table') {
    const desc = element.description ? ` - ${element.description}` : '';
    const rowCount = element.metadata.rowCountEstimate
      ? ` (approx ${element.metadata.rowCountEstimate} rows)`
      : '';
    return `${element.schemaName}.${element.tableName}: table${rowCount}${desc}`;
  }

  // Column
  const desc = element.description ? ` - ${element.description}` : '';
  const fk = element.metadata.isForeignKey
    ? ` [FK -> ${element.metadata.foreignTable}.${element.metadata.foreignColumn}]`
    : '';
  return `${element.schemaName}.${element.tableName}.${element.columnName}: ${element.dataType}${fk}${desc}`;
}

export async function embedSchemaElements(
  elements: SchemaElement[],
  provider: EmbeddingProvider,
  batchSize = 100,
): Promise<Array<{ element: SchemaElement; embedding: number[] }>> {
  const results: Array<{ element: SchemaElement; embedding: number[] }> = [];

  for (let i = 0; i < elements.length; i += batchSize) {
    const batch = elements.slice(i, i + batchSize);
    const texts = batch.map(buildEmbeddingText);
    const embeddings = await provider.embed(texts);

    for (let j = 0; j < batch.length; j++) {
      results.push({ element: batch[j], embedding: embeddings[j] });
    }
  }

  return results;
}
```

**Testing:**
- PostgreSQL connector introspects a test database with 20+ tables and retrieves all tables, columns, types, and foreign key relationships correctly
- MySQL connector introspects the same schema structure via MySQL's `information_schema`
- Schema sync pipeline stores all elements in `schema_elements` with valid vector embeddings (dimension = 1536)
- HNSW index is created and vector similarity search returns relevant results (query "revenue by region" returns revenue/region/amount columns with similarity > 0.7)
- Connection credentials are encrypted at rest and decrypted only when establishing a connection
- UI displays schema tree with tables and columns; search filters work
- Test connection button validates credentials before saving

**Estimated duration:** 3 weeks

---

### Phase 3: Semantic Layer Integration (Cube.js)

**Goal:** Users can connect to a Cube.js deployment, the system syncs semantic model definitions (cubes, measures, dimensions, joins), and stores them with vector embeddings. Semantic models are browsable in the UI.

**What:**
- Define Drizzle schema for `semantic_models` and `semantic_glossary` tables
- Build the Cube.js adapter: fetch model definitions via Cube.js Meta API, map to the internal semantic model format
- Implement semantic model sync: pull cube definitions, store as JSONB, generate embeddings for measures/dimensions
- Build the semantic model browser UI: list models, view measures/dimensions/joins, edit descriptions
- Build the business glossary UI: add/edit terms with synonyms, map to semantic model references
- Generate embeddings for glossary terms

**Design:**

Cube.js adapter:

```typescript
// packages/query-engine/src/semantic/cube-adapter.ts

import type { SemanticModel, SemanticMeasure, SemanticDimension, SemanticJoin } from './types';

interface CubeMetaResponse {
  cubes: Array<{
    name: string;
    title: string;
    description?: string;
    measures: Array<{
      name: string;
      title: string;
      shortTitle: string;
      type: string; // 'count', 'sum', 'avg', etc.
      description?: string;
      aggType: string;
      format?: string;
    }>;
    dimensions: Array<{
      name: string;
      title: string;
      shortTitle: string;
      type: string; // 'string', 'number', 'time', 'boolean'
      description?: string;
      primaryKey?: boolean;
    }>;
    joins: Array<{
      name: string;
      relationship: string; // 'belongsTo', 'hasMany', 'hasOne'
    }>;
  }>;
}

export class CubeAdapter {
  constructor(
    private apiUrl: string,
    private apiToken?: string,
  ) {}

  async fetchModels(): Promise<SemanticModel[]> {
    const response = await fetch(`${this.apiUrl}/v1/meta`, {
      headers: this.apiToken ? { Authorization: `Bearer ${this.apiToken}` } : {},
    });
    const meta: CubeMetaResponse = await response.json();

    return meta.cubes.map((cube) => ({
      name: cube.name,
      displayName: cube.title,
      description: cube.description ?? '',
      sourceType: 'cubejs' as const,
      definition: {
        measures: cube.measures.map((m) => ({
          name: m.name,
          displayName: m.title,
          type: m.type,
          description: m.description ?? '',
          format: m.format,
        })),
        dimensions: cube.dimensions.map((d) => ({
          name: d.name,
          displayName: d.title,
          type: d.type,
          description: d.description ?? '',
          isPrimaryKey: d.primaryKey ?? false,
        })),
        joins: cube.joins.map((j) => ({
          to: j.name,
          relationship: this.mapRelationship(j.relationship),
        })),
      },
    }));
  }

  private mapRelationship(cubeRelationship: string): string {
    const map: Record<string, string> = {
      belongsTo: 'many_to_one',
      hasMany: 'one_to_many',
      hasOne: 'one_to_one',
    };
    return map[cubeRelationship] ?? 'many_to_one';
  }
}
```

Semantic context builder (combines schema, models, and glossary for the LLM):

```typescript
// packages/query-engine/src/pipeline/context-assembler.ts

import type { EmbeddingProvider } from '../embeddings/embedder';
import { db } from '../db';
import { schemaElements, semanticModels, semanticGlossary } from '../db/schema';
import { cosineDistance, desc, sql } from 'drizzle-orm';

export interface SemanticContext {
  schemaElements: Array<{ ref: string; description: string; similarity: number }>;
  semanticModels: Array<{ name: string; definition: unknown; similarity: number }>;
  glossaryTerms: Array<{ term: string; definition: string; mapsTo: unknown; similarity: number }>;
}

export async function assembleContext(
  question: string,
  dataSourceId: string,
  organizationId: string,
  embeddingProvider: EmbeddingProvider,
  options: { schemaLimit?: number; modelLimit?: number; glossaryLimit?: number } = {},
): Promise<SemanticContext> {
  const { schemaLimit = 10, modelLimit = 3, glossaryLimit = 3 } = options;

  // Embed the user's question
  const [questionEmbedding] = await embeddingProvider.embed([question]);

  // Parallel retrieval: schema elements, semantic models, glossary
  const [schemaMatches, modelMatches, glossaryMatches] = await Promise.all([
    db
      .select({
        ref: sql<string>`${schemaElements.schemaName} || '.' || ${schemaElements.tableName} || COALESCE('.' || ${schemaElements.columnName}, '')`,
        description: schemaElements.description,
        similarity: sql<number>`1 - (${schemaElements.embedding} <=> ${JSON.stringify(questionEmbedding)}::vector)`,
      })
      .from(schemaElements)
      .where(sql`${schemaElements.dataSourceId} = ${dataSourceId} AND ${schemaElements.isVisible} = true`)
      .orderBy(sql`${schemaElements.embedding} <=> ${JSON.stringify(questionEmbedding)}::vector`)
      .limit(schemaLimit),

    db
      .select({
        name: semanticModels.name,
        definition: semanticModels.definition,
        similarity: sql<number>`1 - (${semanticModels.embedding} <=> ${JSON.stringify(questionEmbedding)}::vector)`,
      })
      .from(semanticModels)
      .where(sql`${semanticModels.organizationId} = ${organizationId} AND ${semanticModels.isPublished} = true`)
      .orderBy(sql`${semanticModels.embedding} <=> ${JSON.stringify(questionEmbedding)}::vector`)
      .limit(modelLimit),

    db
      .select({
        term: semanticGlossary.term,
        definition: semanticGlossary.definition,
        mapsTo: semanticGlossary.mapsTo,
        similarity: sql<number>`1 - (${semanticGlossary.embedding} <=> ${JSON.stringify(questionEmbedding)}::vector)`,
      })
      .from(semanticGlossary)
      .where(sql`${semanticGlossary.organizationId} = ${organizationId}`)
      .orderBy(sql`${semanticGlossary.embedding} <=> ${JSON.stringify(questionEmbedding)}::vector`)
      .limit(glossaryLimit),
  ]);

  return {
    schemaElements: schemaMatches,
    semanticModels: modelMatches,
    glossaryTerms: glossaryMatches,
  };
}
```

**Testing:**
- Cube.js adapter fetches models from a running Cube.js instance and correctly maps all measures, dimensions, and joins
- Semantic model sync stores definitions as JSONB and generates valid embeddings
- Vector similarity search across semantic models returns relevant models for domain-specific questions
- Glossary term "ARR" maps to the correct semantic model measure when a user asks "Show me ARR by quarter"
- Combined context assembly (schema + models + glossary) returns a ranked, deduplicated set of context items
- Semantic model browser UI displays all synced models with their measures and dimensions

**Estimated duration:** 2 weeks

---

### Phase 4: NL-to-SQL Query Engine (Core Pipeline)

**Goal:** Users can type a natural language question and receive a SQL query, executed results, and a data table. This is the core differentiator — the full pipeline from question to answer.

**What:**
- Build the query generation pipeline: context assembly -> prompt construction -> LLM call -> SQL extraction -> validation
- Implement LLM provider abstraction (Anthropic Claude, OpenAI GPT-4)
- Build the SQL validator: parse generated SQL, check for prohibited operations (DROP, DELETE, INSERT, UPDATE), verify table/column references against schema metadata
- Build the query executor: run validated SQL against the source database with timeout and row limits
- Implement query error correction: on SQL error, send the error message back to the LLM with the original question for auto-correction (up to 2 retries)
- Define Drizzle schema for `conversations`, `messages`, `queries` tables
- Build the conversation UI: chat interface with message history, SQL display, result table

**Design:**

Query generation pipeline:

```typescript
// packages/query-engine/src/pipeline/query-generator.ts

import type { SemanticContext } from './context-assembler';
import type { LLMProvider } from '../llm/provider';

interface GenerationResult {
  sql: string;
  confidence: number;
  explanation: string;
  referencedModels: string[];
  referencedMeasures: string[];
  referencedDimensions: string[];
}

export async function generateSQL(
  question: string,
  context: SemanticContext,
  conversationHistory: Array<{ role: 'user' | 'assistant'; content: string }>,
  dialect: string,
  llmProvider: LLMProvider,
): Promise<GenerationResult> {
  const systemPrompt = buildSystemPrompt(context, dialect);
  const messages = [
    { role: 'system' as const, content: systemPrompt },
    ...conversationHistory,
    { role: 'user' as const, content: question },
  ];

  const response = await llmProvider.chat(messages, {
    temperature: 0,
    maxTokens: 2000,
    responseFormat: {
      type: 'json',
      schema: {
        type: 'object',
        properties: {
          sql: { type: 'string' },
          confidence: { type: 'number', minimum: 0, maximum: 1 },
          explanation: { type: 'string' },
          referenced_models: { type: 'array', items: { type: 'string' } },
          referenced_measures: { type: 'array', items: { type: 'string' } },
          referenced_dimensions: { type: 'array', items: { type: 'string' } },
        },
        required: ['sql', 'confidence', 'explanation'],
      },
    },
  });

  const parsed = JSON.parse(response.content);
  return {
    sql: parsed.sql,
    confidence: parsed.confidence,
    explanation: parsed.explanation,
    referencedModels: parsed.referenced_models ?? [],
    referencedMeasures: parsed.referenced_measures ?? [],
    referencedDimensions: parsed.referenced_dimensions ?? [],
  };
}

function buildSystemPrompt(context: SemanticContext, dialect: string): string {
  const schemaSection = context.schemaElements
    .map((e) => `- ${e.ref}: ${e.description ?? 'no description'}`)
    .join('\n');

  const modelsSection = context.semanticModels
    .map((m) => {
      const def = m.definition as { measures?: unknown[]; dimensions?: unknown[] };
      return `Model "${m.name}":\n  Measures: ${JSON.stringify(def.measures)}\n  Dimensions: ${JSON.stringify(def.dimensions)}`;
    })
    .join('\n\n');

  const glossarySection = context.glossaryTerms
    .map((g) => `- "${g.term}": ${g.definition} (maps to: ${JSON.stringify(g.mapsTo)})`)
    .join('\n');

  return `You are a SQL expert that generates accurate ${dialect} queries based on user questions.

## Available Schema Context
${schemaSection}

## Semantic Models (Governed Metrics)
${modelsSection}

## Business Glossary
${glossarySection}

## Rules
1. ONLY use tables and columns listed in the schema context above.
2. When a semantic model defines a measure (e.g., "total_revenue" = SUM(amount)), use the measure's SQL expression exactly.
3. Generate valid ${dialect} SQL. Do not use features from other SQL dialects.
4. For time-based questions, use appropriate date functions for ${dialect}.
5. Always include a confidence score (0.0 to 1.0) reflecting how certain you are the SQL answers the question correctly.
6. If the question is ambiguous, generate the most likely interpretation and explain your assumption.
7. NEVER generate INSERT, UPDATE, DELETE, DROP, ALTER, TRUNCATE, or any DDL/DML statements.
8. Respond in the specified JSON format.`;
}
```

SQL validator:

```typescript
// packages/query-engine/src/pipeline/query-validator.ts

const PROHIBITED_PATTERNS = [
  /\b(INSERT|UPDATE|DELETE|DROP|ALTER|TRUNCATE|CREATE|GRANT|REVOKE)\b/i,
  /\bINTO\s+OUTFILE\b/i,
  /\bLOAD\s+DATA\b/i,
  /;\s*\w/,  // Multiple statements (SQL injection prevention)
];

export interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
  sanitizedSql: string;
}

export function validateGeneratedSQL(
  sql: string,
  knownTables: Set<string>,
  knownColumns: Map<string, Set<string>>,
): ValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];

  // Check for prohibited operations
  for (const pattern of PROHIBITED_PATTERNS) {
    if (pattern.test(sql)) {
      errors.push(`Prohibited SQL operation detected: ${pattern.source}`);
    }
  }

  // Verify the query is a SELECT statement
  const trimmed = sql.trim();
  if (!/^(SELECT|WITH)\b/i.test(trimmed)) {
    errors.push('Only SELECT queries are permitted');
  }

  // Add LIMIT if not present (safety net)
  let sanitizedSql = trimmed;
  if (!/\bLIMIT\b/i.test(sanitizedSql)) {
    sanitizedSql = `${sanitizedSql.replace(/;?\s*$/, '')} LIMIT 10000`;
    warnings.push('Added LIMIT 10000 to prevent unbounded result sets');
  }

  return {
    valid: errors.length === 0,
    errors,
    warnings,
    sanitizedSql,
  };
}
```

**Testing:**
- End-to-end: user types "What was total revenue last quarter?" -> system generates valid SQL -> executes against test database -> returns correct numeric result
- Confidence score is > 0.8 for simple single-table queries on a well-described schema
- SQL validator rejects all prohibited operations (INSERT, DROP, multi-statement injection)
- Query correction: when LLM generates a column typo, the system auto-corrects within 2 retries and returns the correct result
- Conversation history is maintained: "now break that down by region" references the previous query's table and adds GROUP BY
- LLM provider abstraction works with both Anthropic Claude and OpenAI GPT-4 (tested via mock and live API calls)
- Queries with confidence < 0.5 display a warning banner to the user

**Estimated duration:** 3 weeks

---

### Phase 5: Visualization Engine and Chart Rendering

**Goal:** Query results are automatically visualized with an appropriate chart type. Users can change chart type, customize axes, and save visualizations.

**What:**
- Build the chart type recommender: analyse result column types and cardinality to suggest the best visualization
- Implement 8 core chart components using ECharts: table, bar, line, area, pie, scatter, number/KPI, stacked bar
- Build the chart configuration UI: axis selection, color palette, sorting, legend toggle
- Define Drizzle schema for `visualizations` table
- Connect the conversation flow: after query execution, auto-recommend a chart type, render inline
- Implement "save visualization" flow

**Design:**

Chart type recommender:

```typescript
// packages/chart-engine/src/recommender.ts

interface ResultColumn {
  name: string;
  type: 'string' | 'number' | 'date' | 'boolean';
  distinctCount?: number;
  sampleValues?: unknown[];
}

interface ChartRecommendation {
  chartType: string;
  confidence: number;
  config: Record<string, unknown>;
  reasoning: string;
}

export function recommendChartType(
  columns: ResultColumn[],
  rowCount: number,
): ChartRecommendation {
  const stringCols = columns.filter((c) => c.type === 'string');
  const numberCols = columns.filter((c) => c.type === 'number');
  const dateCols = columns.filter((c) => c.type === 'date');

  // Single number result -> KPI card
  if (rowCount === 1 && numberCols.length === 1 && stringCols.length === 0) {
    return {
      chartType: 'number',
      confidence: 0.95,
      config: { value: numberCols[0].name },
      reasoning: 'Single numeric value best displayed as a KPI card',
    };
  }

  // Time series: date column + number column(s) -> line chart
  if (dateCols.length === 1 && numberCols.length >= 1) {
    return {
      chartType: 'line',
      confidence: 0.9,
      config: {
        xAxis: dateCols[0].name,
        yAxis: numberCols.map((c) => c.name),
        granularity: 'auto',
      },
      reasoning: 'Time series data with numeric measures displayed as a line chart',
    };
  }

  // Categorical breakdown: string column + number column -> bar chart
  if (stringCols.length === 1 && numberCols.length >= 1 && rowCount <= 30) {
    return {
      chartType: 'bar',
      confidence: 0.85,
      config: {
        xAxis: stringCols[0].name,
        yAxis: numberCols[0].name,
        sort: 'desc',
      },
      reasoning: 'Categorical breakdown with manageable cardinality',
    };
  }

  // Two number columns -> scatter plot
  if (numberCols.length === 2 && stringCols.length === 0) {
    return {
      chartType: 'scatter',
      confidence: 0.75,
      config: { xAxis: numberCols[0].name, yAxis: numberCols[1].name },
      reasoning: 'Two numeric dimensions suggest correlation analysis',
    };
  }

  // Proportion: string + single number with low cardinality -> pie
  if (stringCols.length === 1 && numberCols.length === 1 && rowCount <= 8) {
    return {
      chartType: 'pie',
      confidence: 0.7,
      config: {
        dimension: stringCols[0].name,
        measure: numberCols[0].name,
      },
      reasoning: 'Low-cardinality proportion data displayed as pie chart',
    };
  }

  // Default: table
  return {
    chartType: 'table',
    confidence: 0.6,
    config: { columns: columns.map((c) => c.name) },
    reasoning: 'Complex or high-cardinality data displayed as a table',
  };
}
```

**Testing:**
- Recommender correctly suggests: KPI for single-number result, line for time series, bar for categorical breakdown, pie for low-cardinality proportions, table as fallback
- All 8 chart types render correctly with test data
- Chart configuration changes (swap axes, change color palette) update the visualization in real time
- Saved visualizations persist and can be retrieved from the conversation history
- Charts are responsive and render correctly on viewport widths from 320px to 1920px
- Chart export to PNG via ECharts' built-in export function

**Estimated duration:** 2 weeks

---

### Phase 6: Dashboards and Sharing

**Goal:** Users can create dashboards from saved visualizations, arrange them in a grid layout, add dashboard-level filters, and share dashboards via links or role-based access.

**What:**
- Define Drizzle schema for `dashboards` table (with JSONB layout and sharing config)
- Build the dashboard editor: drag-and-drop grid layout (react-grid-layout), add/remove visualization widgets
- Build dashboard-level filter controls (dimension dropdowns, date range pickers) that propagate to all widgets
- Implement sharing: generate shareable links with optional expiry, role-based access control, embed tokens
- Build the dashboard viewer: read-only view with auto-refresh, filter interaction
- Implement dashboard URL routing: `/dashboards/:id`, `/embed/:token`

**Design:**

Dashboard layout using react-grid-layout:

```typescript
// apps/web/src/components/dashboard/DashboardEditor.tsx

import { Responsive, WidthProvider, type Layout } from 'react-grid-layout';
import type { Dashboard, DashboardWidget } from '@/lib/db/schema';

const ResponsiveGrid = WidthProvider(Responsive);

interface DashboardEditorProps {
  dashboard: Dashboard;
  widgets: DashboardWidget[];
  onLayoutChange: (layout: Layout[]) => void;
  onAddWidget: () => void;
  onRemoveWidget: (widgetId: string) => void;
}

export function DashboardEditor({
  dashboard,
  widgets,
  onLayoutChange,
  onAddWidget,
  onRemoveWidget,
}: DashboardEditorProps) {
  const layouts = widgets.map((w) => ({
    i: w.visualizationId,
    x: w.position.x,
    y: w.position.y,
    w: w.position.w,
    h: w.position.h,
    minW: 2,
    minH: 2,
  }));

  return (
    <div className="dashboard-editor">
      <DashboardToolbar
        title={dashboard.title}
        onAddWidget={onAddWidget}
        onSave={() => { /* persist layout */ }}
        onShare={() => { /* open share dialog */ }}
      />
      <DashboardFilters
        filters={dashboard.config.filters ?? []}
        onChange={(filters) => { /* update filter state */ }}
      />
      <ResponsiveGrid
        layouts={{ lg: layouts }}
        breakpoints={{ lg: 1200, md: 996, sm: 768 }}
        cols={{ lg: 12, md: 8, sm: 4 }}
        rowHeight={80}
        onLayoutChange={(layout) => onLayoutChange(layout)}
        draggableHandle=".widget-drag-handle"
      >
        {widgets.map((widget) => (
          <div key={widget.visualizationId}>
            <VisualizationWidget
              visualizationId={widget.visualizationId}
              onRemove={() => onRemoveWidget(widget.visualizationId)}
            />
          </div>
        ))}
      </ResponsiveGrid>
    </div>
  );
}
```

Share link generation:

```typescript
// apps/web/src/lib/trpc/routers/dashboard.ts — share procedure

import { randomBytes } from 'crypto';
import { TRPCError } from '@trpc/server';

export const shareDashboard = protectedProcedure
  .input(z.object({
    dashboardId: z.string().uuid(),
    shareType: z.enum(['link', 'embed']),
    accessLevel: z.enum(['view', 'edit']).default('view'),
    expiresInDays: z.number().int().positive().optional(),
    allowedDomains: z.array(z.string()).optional(), // for embed only
  }))
  .mutation(async ({ ctx, input }) => {
    const dashboard = await ctx.db.query.dashboards.findFirst({
      where: (d, { eq, and }) =>
        and(eq(d.id, input.dashboardId), eq(d.organizationId, ctx.user.organizationId)),
    });

    if (!dashboard) throw new TRPCError({ code: 'NOT_FOUND' });

    const token = randomBytes(32).toString('base64url');
    const expiresAt = input.expiresInDays
      ? new Date(Date.now() + input.expiresInDays * 86400000)
      : null;

    // Merge into the dashboard's sharing JSONB
    const updatedSharing = {
      ...dashboard.sharing,
      [input.shareType === 'link' ? 'public_link' : 'embed']: {
        token,
        accessLevel: input.accessLevel,
        expiresAt: expiresAt?.toISOString() ?? null,
        allowedDomains: input.allowedDomains ?? [],
        createdBy: ctx.user.id,
        createdAt: new Date().toISOString(),
      },
    };

    await ctx.db
      .update(dashboards)
      .set({ sharing: updatedSharing, updatedAt: new Date() })
      .where(eq(dashboards.id, input.dashboardId));

    const baseUrl = process.env.NEXTAUTH_URL;
    return {
      url: input.shareType === 'link'
        ? `${baseUrl}/dashboards/shared/${token}`
        : `${baseUrl}/embed/${token}`,
      token,
      expiresAt,
    };
  });
```

**Testing:**
- Dashboard editor: drag widgets to rearrange, resize widgets, layout persists on reload
- Dashboard filters: selecting a region filter updates all chart widgets to show filtered data
- Shared link: unauthenticated user can view a dashboard via its share link
- Expired share links return a 403 with an appropriate message
- Embed token: dashboard renders in an iframe at an allowed domain; requests from disallowed domains are rejected
- Role-based access: viewer cannot edit dashboard layout; editor can
- Auto-refresh: dashboard polls for updated data at the configured interval

**Estimated duration:** 3 weeks

---

### Phase 7: Automated Insight Narration

**Goal:** Every visualization is accompanied by an LLM-generated narrative explaining what the data shows, trends, anomalies, and actionable implications. This is the feature that makes dashboards accessible to non-analysts (README.md: "AI-Native Opportunity #3").

**What:**
- Build the insight engine (`packages/insight-engine/`): analyze query results, generate narrative text via LLM
- Implement trend detection: compare current period to previous period, calculate percentage change
- Implement anomaly flagging: identify statistical outliers in the result set
- Generate natural language narratives: "Revenue increased 23% QoQ, driven primarily by APAC region (+41%). North America remained flat. The EMEA decline (-8%) is notable and warrants investigation."
- Display insight text below each visualization in conversations and dashboards
- Store insight text in the `visualizations.insight_text` column

**Design:**

Insight generation pipeline:

```typescript
// packages/insight-engine/src/narrator.ts

import type { LLMProvider } from '@nlbi/query-engine/llm/provider';

interface InsightInput {
  question: string;
  sql: string;
  columns: Array<{ name: string; type: string }>;
  data: Record<string, unknown>[];
  chartType: string;
  previousPeriodData?: Record<string, unknown>[];
}

interface Insight {
  narrative: string;
  highlights: Array<{
    type: 'trend' | 'anomaly' | 'comparison' | 'recommendation';
    text: string;
    severity: 'info' | 'warning' | 'critical';
  }>;
}

export async function generateInsight(
  input: InsightInput,
  llmProvider: LLMProvider,
): Promise<Insight> {
  const prompt = `Analyze the following query results and provide a concise business insight.

Question asked: "${input.question}"
Chart type: ${input.chartType}
Columns: ${JSON.stringify(input.columns)}
Data (${input.data.length} rows): ${JSON.stringify(input.data.slice(0, 50))}
${input.previousPeriodData ? `Previous period data: ${JSON.stringify(input.previousPeriodData.slice(0, 50))}` : ''}

Respond in JSON with:
- "narrative": A 2-4 sentence business-friendly summary of the key findings. Lead with the most important insight. Mention specific numbers and percentages.
- "highlights": Array of notable findings, each with:
  - "type": "trend" | "anomaly" | "comparison" | "recommendation"
  - "text": One sentence describing the finding
  - "severity": "info" | "warning" | "critical"

Focus on what is actionable. Do not describe the chart type or repeat the question.`;

  const response = await llmProvider.chat(
    [{ role: 'user', content: prompt }],
    { temperature: 0.3, maxTokens: 500, responseFormat: { type: 'json' } },
  );

  return JSON.parse(response.content) as Insight;
}
```

**Testing:**
- Insight generated for a revenue-by-region query mentions the top and bottom performing regions by name
- Trend detection: when previous period data is provided, the narrative includes percentage change
- Anomaly flagging: a 10x spike in a single region is flagged with severity "warning"
- Insight text renders correctly below the chart in both conversation and dashboard views
- Insight generation completes within 3 seconds (LLM call + formatting)
- Insight is persisted and does not re-generate on page refresh

**Estimated duration:** 2 weeks

---

### Phase 8: Multi-Turn Conversation and Query Decomposition

**Goal:** The system handles complex multi-turn conversations with full context retention and decomposes complex analytical questions into multi-step SQL queries. This addresses the research finding that "every current tool treats NL queries as stateless by default" (features.md).

**What:**
- Implement conversation context accumulation: track active filters, referenced tables, resolved ambiguities in `conversations.context` JSONB
- Build follow-up query resolution: "now break that down by region" -> system infers the base table, measures, and adds GROUP BY region
- Implement query decomposition (DIN-SQL approach): break complex multi-join questions into sub-queries, execute sequentially, combine results
- Handle clarification requests: when the LLM confidence is below a threshold, ask a clarifying question before generating SQL
- Implement "undo" and "modify" conversation commands: "go back to the previous version", "change the date range to Q2"

**Design:**

Conversation context manager:

```typescript
// packages/query-engine/src/pipeline/conversation-context.ts

export interface ConversationContext {
  activeFilters: Record<string, unknown>;
  referencedTables: string[];
  referencedMeasures: string[];
  referencedDimensions: string[];
  clarifications: Array<{ term: string; resolvedTo: string }>;
  lastQuery?: {
    sql: string;
    columns: string[];
    dataSourceId: string;
  };
}

export function updateContext(
  current: ConversationContext,
  queryResult: {
    sql: string;
    referencedModels: string[];
    referencedMeasures: string[];
    referencedDimensions: string[];
    columns: string[];
    dataSourceId: string;
  },
): ConversationContext {
  return {
    activeFilters: current.activeFilters,
    referencedTables: [
      ...new Set([...current.referencedTables, ...queryResult.referencedModels]),
    ],
    referencedMeasures: [
      ...new Set([...current.referencedMeasures, ...queryResult.referencedMeasures]),
    ],
    referencedDimensions: [
      ...new Set([...current.referencedDimensions, ...queryResult.referencedDimensions]),
    ],
    clarifications: current.clarifications,
    lastQuery: {
      sql: queryResult.sql,
      columns: queryResult.columns,
      dataSourceId: queryResult.dataSourceId,
    },
  };
}

export function isFollowUp(question: string): boolean {
  const followUpPatterns = [
    /^(now|and|also|then|next)\b/i,
    /^(break|split|group|segment)\s+(that|this|it)\b/i,
    /^(show|display)\s+(that|this|it)\b/i,
    /^(exclude|remove|filter|add)\b/i,
    /^(compare|vs|versus)\b/i,
    /^(what about|how about)\b/i,
    /^(same|similar)\s+(but|for|with)\b/i,
  ];
  return followUpPatterns.some((p) => p.test(question.trim()));
}
```

Query decomposer (DIN-SQL approach):

```typescript
// packages/query-engine/src/pipeline/decomposer.ts

import type { LLMProvider } from '../llm/provider';
import type { SemanticContext } from './context-assembler';

interface DecompositionStep {
  stepNumber: number;
  subQuestion: string;
  dependsOn: number[];  // step numbers this depends on
  sql?: string;
  result?: unknown;
}

export async function decomposeQuery(
  question: string,
  context: SemanticContext,
  dialect: string,
  llmProvider: LLMProvider,
): Promise<DecompositionStep[]> {
  const prompt = `Analyze this analytical question and determine if it needs to be broken into sub-queries.

Question: "${question}"

Available schema: ${JSON.stringify(context.schemaElements.map((e) => e.ref))}
Available metrics: ${JSON.stringify(context.semanticModels.map((m) => m.name))}

If the question requires multiple steps (e.g., comparing periods, calculating ratios from different tables, or combining results from independent queries), decompose it.

Respond in JSON:
{
  "needs_decomposition": boolean,
  "steps": [
    {
      "step_number": 1,
      "sub_question": "What is the total revenue for Q1 2026?",
      "depends_on": [],
      "reasoning": "First, get the base revenue figure"
    },
    {
      "step_number": 2,
      "sub_question": "What is the total revenue for Q4 2025?",
      "depends_on": [],
      "reasoning": "Get the comparison period revenue"
    },
    {
      "step_number": 3,
      "sub_question": "Calculate the percentage change between step 1 and step 2",
      "depends_on": [1, 2],
      "reasoning": "Compute the QoQ change using both results"
    }
  ]
}

If the question is simple enough for a single query, return {"needs_decomposition": false, "steps": []} and the caller will generate SQL directly.`;

  const response = await llmProvider.chat(
    [{ role: 'user', content: prompt }],
    { temperature: 0, maxTokens: 1000, responseFormat: { type: 'json' } },
  );

  const parsed = JSON.parse(response.content);
  return parsed.needs_decomposition ? parsed.steps : [];
}
```

**Testing:**
- Follow-up detection: "now break that down by region" is detected as a follow-up; "What was revenue last year?" is not
- Context retention: after asking "revenue by region", a follow-up "exclude APAC" produces SQL with `WHERE region != 'APAC'` added to the previous query's base
- Query decomposition: "Compare Q1 2026 revenue to Q4 2025" decomposes into two sub-queries plus a comparison step
- Clarification: when the LLM confidence is below 0.5, the system responds with a clarifying question instead of SQL
- Undo: "go back" restores the previous query and visualization state
- 10-turn conversation maintains coherent context without degradation

**Estimated duration:** 3 weeks

---

### Phase 9: Alerts, Scheduled Reports, and Notifications

**Goal:** Users can set up metric alerts (threshold, anomaly, no-data) and scheduled reports delivered via Slack and email.

**What:**
- Define Drizzle schema for `alerts` table (JSONB config + state pattern from data model suggestion 3)
- Build the alert scheduler: cron-based evaluation of alert conditions using node-cron or BullMQ
- Implement alert condition types: threshold_above, threshold_below, change_percent, anomaly, no_data
- Build Slack and email notification channels
- Create the alerts management UI: create, edit, view history, pause/resume
- Implement scheduled dashboard snapshots: render dashboard to image, send via email/Slack on a schedule

**Design:**

Alert evaluator:

```typescript
// apps/web/src/lib/alerts/evaluator.ts

import type { DatabaseConnector } from '@nlbi/query-engine/connectors/base';

interface AlertConfig {
  queryTemplate: string;
  condition: {
    type: 'threshold_above' | 'threshold_below' | 'change_percent' | 'anomaly' | 'no_data';
    value: number;
  };
  notifications: Array<{
    channel: 'slack' | 'email';
    webhookUrl?: string;
    recipients?: string[];
  }>;
}

interface AlertState {
  isActive: boolean;
  lastEvaluatedAt: string | null;
  lastTriggeredAt: string | null;
  lastValue: number | null;
  triggerCount: number;
  consecutiveTriggers: number;
}

interface EvaluationResult {
  triggered: boolean;
  currentValue: number | null;
  message: string;
}

export async function evaluateAlert(
  config: AlertConfig,
  state: AlertState,
  connector: DatabaseConnector,
): Promise<EvaluationResult> {
  const result = await connector.executeQuery(config.queryTemplate, [], 30000);

  if (result.rowCount === 0) {
    if (config.condition.type === 'no_data') {
      return { triggered: true, currentValue: null, message: 'No data returned by alert query' };
    }
    return { triggered: false, currentValue: null, message: 'No data' };
  }

  const firstCol = result.columns[0];
  const currentValue = Number(result.rows[0][firstCol.name]);

  switch (config.condition.type) {
    case 'threshold_above':
      return {
        triggered: currentValue > config.condition.value,
        currentValue,
        message: `Current value ${currentValue} ${currentValue > config.condition.value ? 'exceeds' : 'is below'} threshold ${config.condition.value}`,
      };

    case 'threshold_below':
      return {
        triggered: currentValue < config.condition.value,
        currentValue,
        message: `Current value ${currentValue} ${currentValue < config.condition.value ? 'is below' : 'is above'} threshold ${config.condition.value}`,
      };

    case 'change_percent': {
      if (state.lastValue === null) {
        return { triggered: false, currentValue, message: 'No previous value for comparison' };
      }
      const changePercent = ((currentValue - state.lastValue) / Math.abs(state.lastValue)) * 100;
      return {
        triggered: Math.abs(changePercent) > config.condition.value,
        currentValue,
        message: `Value changed ${changePercent.toFixed(1)}% (from ${state.lastValue} to ${currentValue}). Threshold: +/-${config.condition.value}%`,
      };
    }

    default:
      return { triggered: false, currentValue, message: 'Unknown condition type' };
  }
}
```

**Testing:**
- Alert with threshold_above triggers when the query returns a value above the threshold and does not trigger below
- change_percent alert detects a 25% increase when threshold is 20%
- no_data alert triggers when the query returns zero rows
- Slack notification sends a formatted message with alert name, current value, threshold, and a link to the dashboard
- Email notification sends an HTML-formatted alert summary
- Alert history records all evaluations with timestamps, values, and trigger status
- Paused alerts are not evaluated by the scheduler
- Scheduler handles 100+ active alerts without blocking

**Estimated duration:** 2 weeks

---

### Phase 10: Schema Change Detection and Self-Healing Dashboards

**Goal:** When a connected database's schema changes (renamed columns, removed tables), the system detects the change, identifies affected queries and dashboards, and proposes automatic repairs using embedding similarity.

**What:**
- Build the schema diff engine: compare current schema sync against the previous sync, detect additions, removals, renames, type changes
- Implement rename detection: use embedding similarity between old and new column names/descriptions to identify likely renames (threshold: 0.85 similarity)
- Build the impact analysis: identify all queries, visualizations, and dashboards that reference affected schema elements
- Implement auto-repair proposals: generate SQL rewrites that replace old references with new ones
- Build the schema change notification UI: show changes, proposed repairs, and allow users to accept/reject
- Store changes in `schema_change_log` table

**Design:**

Schema diff and rename detection:

```typescript
// packages/query-engine/src/semantic/schema-diff.ts

import type { SchemaElement } from '../connectors/base';
import type { EmbeddingProvider } from '../embeddings/embedder';

interface SchemaChange {
  changeType: 'table_added' | 'table_removed' | 'table_renamed' |
              'column_added' | 'column_removed' | 'column_renamed' | 'column_type_changed';
  details: {
    schema: string;
    table: string;
    oldColumn?: string;
    newColumn?: string;
    oldType?: string;
    newType?: string;
    similarityScore?: number;
  };
}

export async function detectSchemaChanges(
  previousElements: SchemaElement[],
  currentElements: SchemaElement[],
  embeddingProvider: EmbeddingProvider,
  renameThreshold = 0.85,
): Promise<SchemaChange[]> {
  const changes: SchemaChange[] = [];

  const prevTables = new Set(previousElements.filter((e) => e.elementType === 'table')
    .map((e) => `${e.schemaName}.${e.tableName}`));
  const currTables = new Set(currentElements.filter((e) => e.elementType === 'table')
    .map((e) => `${e.schemaName}.${e.tableName}`));

  // Find removed tables
  const removedTables = [...prevTables].filter((t) => !currTables.has(t));
  const addedTables = [...currTables].filter((t) => !prevTables.has(t));

  // Try to match removed tables to added tables via embedding similarity
  if (removedTables.length > 0 && addedTables.length > 0) {
    const allTexts = [...removedTables, ...addedTables];
    const embeddings = await embeddingProvider.embed(allTexts);
    const removedEmbeddings = embeddings.slice(0, removedTables.length);
    const addedEmbeddings = embeddings.slice(removedTables.length);

    for (let i = 0; i < removedTables.length; i++) {
      let bestMatch = -1;
      let bestSimilarity = 0;

      for (let j = 0; j < addedTables.length; j++) {
        const similarity = cosineSimilarity(removedEmbeddings[i], addedEmbeddings[j]);
        if (similarity > bestSimilarity) {
          bestSimilarity = similarity;
          bestMatch = j;
        }
      }

      if (bestSimilarity >= renameThreshold && bestMatch >= 0) {
        const [schema, oldTable] = removedTables[i].split('.');
        const [, newTable] = addedTables[bestMatch].split('.');
        changes.push({
          changeType: 'table_renamed',
          details: { schema, table: oldTable, newColumn: newTable, similarityScore: bestSimilarity },
        });
      } else {
        const [schema, table] = removedTables[i].split('.');
        changes.push({ changeType: 'table_removed', details: { schema, table } });
      }
    }
  }

  // Repeat similar logic for column-level changes within each surviving table...
  // (Column rename detection uses the same embedding similarity approach)

  return changes;
}

function cosineSimilarity(a: number[], b: number[]): number {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return dot / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

**Testing:**
- Column rename detection: renaming `total_amt` to `total_amount` is detected as a rename with similarity > 0.9
- Table removal is detected and all affected queries are identified
- Auto-repair: a query referencing `total_amt` is rewritten to use `total_amount` after user accepts the proposal
- Dashboard self-healing: a dashboard with 5 widgets where 2 reference a renamed column shows a repair notification with 2 proposed fixes
- False positive rate: randomly named new columns are not matched to removed columns (similarity below threshold)
- Schema change log records all detected changes with timestamps

**Estimated duration:** 2 weeks

---

### Phase 11: Embeddable Widget SDK and Public API

**Goal:** SaaS companies can embed the NL query widget in their products via a JavaScript SDK with signed-URL authentication. A REST API enables programmatic access.

**What:**
- Build the embed SDK (`packages/embed-sdk/`): lightweight JavaScript bundle that renders the NL query widget in an iframe or Web Component
- Implement signed-URL authentication for embeds: HMAC-signed tokens with organization, expiry, and allowed data sources
- Build the REST API: query execution, conversation management, dashboard retrieval (OpenAPI-documented)
- Implement rate limiting per API key
- Build the API key management UI within organization settings
- Create embed configuration UI: select data sources, customize branding (colors, logo), generate embed code

**Design:**

Embed SDK:

```typescript
// packages/embed-sdk/src/index.ts

export interface NLBIEmbedConfig {
  container: HTMLElement | string;
  token: string;
  apiUrl?: string;
  theme?: 'light' | 'dark' | 'auto';
  primaryColor?: string;
  placeholder?: string;
  showHistory?: boolean;
  maxHeight?: string;
  onQuery?: (query: { question: string; sql: string; confidence: number }) => void;
  onError?: (error: { code: string; message: string }) => void;
}

export function createNLBIEmbed(config: NLBIEmbedConfig): NLBIEmbed {
  const container = typeof config.container === 'string'
    ? document.getElementById(config.container)
    : config.container;

  if (!container) throw new Error('NLBI: Container element not found');

  const iframe = document.createElement('iframe');
  const params = new URLSearchParams({
    token: config.token,
    theme: config.theme ?? 'auto',
    primaryColor: config.primaryColor ?? '#2563eb',
    placeholder: config.placeholder ?? 'Ask a question about your data...',
    showHistory: String(config.showHistory ?? true),
  });

  iframe.src = `${config.apiUrl ?? 'https://app.nlbi.dev'}/embed/widget?${params}`;
  iframe.style.cssText = `
    width: 100%; border: none; border-radius: 8px;
    min-height: 400px; max-height: ${config.maxHeight ?? '800px'};
  `;

  container.appendChild(iframe);

  // Listen for postMessage events from the iframe
  window.addEventListener('message', (event) => {
    if (event.source !== iframe.contentWindow) return;
    const { type, payload } = event.data;
    if (type === 'nlbi:query' && config.onQuery) config.onQuery(payload);
    if (type === 'nlbi:error' && config.onError) config.onError(payload);
  });

  return {
    destroy: () => container.removeChild(iframe),
    setTheme: (theme: 'light' | 'dark') => {
      iframe.contentWindow?.postMessage({ type: 'nlbi:setTheme', payload: { theme } }, '*');
    },
  };
}

export interface NLBIEmbed {
  destroy(): void;
  setTheme(theme: 'light' | 'dark'): void;
}
```

**Testing:**
- Embed SDK renders the NL query widget in a third-party HTML page
- Signed token with expired timestamp returns 401
- Signed token with wrong HMAC signature returns 403
- Embed widget can only query data sources specified in the token
- REST API: POST /api/v1/query with API key returns query result in JSON
- Rate limiting: 101st request within a 1-minute window returns 429
- API key with `read:queries` scope cannot create dashboards
- OpenAPI spec validates against the Swagger validator

**Estimated duration:** 3 weeks

---

### Phase 12: Production Hardening, Deployment, and Documentation

**Goal:** Production-ready deployment with comprehensive documentation, CI/CD, monitoring, and performance optimization.

**What:**
- Implement query result caching: SHA-256 hash of (SQL + data source + params), serve cached results for identical queries within a TTL
- Set up PostgreSQL connection pooling (PgBouncer or built-in pool) for production loads
- Implement request-level tracing (OpenTelemetry) for the full query pipeline: context assembly -> LLM call -> SQL execution
- Build Prometheus metrics: query latency, LLM token usage, cache hit rate, error rate, active connections
- Set up CI/CD: GitHub Actions for lint, type check, unit tests, integration tests, Docker image build
- Write self-hosting documentation: Docker Compose production guide, environment variables, backup procedures
- Write API documentation: OpenAPI spec, SDK quickstart, embedding guide
- Build the Kubernetes Helm chart for enterprise deployments
- Implement database backup and migration safety checks
- Security hardening: CSP headers, CORS configuration, SQL injection double-checking, secrets rotation

**Design:**

OpenTelemetry tracing for the query pipeline:

```typescript
// packages/query-engine/src/pipeline/tracing.ts

import { trace, SpanStatusCode, type Span } from '@opentelemetry/api';

const tracer = trace.getTracer('nlbi-query-engine');

export async function tracedQueryPipeline<T>(
  name: string,
  attributes: Record<string, string | number | boolean>,
  fn: (span: Span) => Promise<T>,
): Promise<T> {
  return tracer.startActiveSpan(name, async (span) => {
    try {
      for (const [key, value] of Object.entries(attributes)) {
        span.setAttribute(key, value);
      }
      const result = await fn(span);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error instanceof Error ? error.message : 'Unknown error',
      });
      span.recordException(error instanceof Error ? error : new Error(String(error)));
      throw error;
    } finally {
      span.end();
    }
  });
}

// Usage in the query pipeline:
// await tracedQueryPipeline('nlbi.context_assembly', { dataSourceId, questionLength: question.length }, async (span) => {
//   const context = await assembleContext(...);
//   span.setAttribute('nlbi.schema_matches', context.schemaElements.length);
//   span.setAttribute('nlbi.model_matches', context.semanticModels.length);
//   return context;
// });
```

Docker Compose production configuration:

```yaml
# docker/docker-compose.yml (production)
services:
  db:
    image: pgvector/pgvector:pg16
    restart: always
    environment:
      POSTGRES_DB: nlbi
      POSTGRES_USER: nlbi
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./postgresql.conf:/etc/postgresql/postgresql.conf
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    deploy:
      resources:
        limits:
          memory: 2G
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nlbi"]
      interval: 10s
      timeout: 5s
      retries: 5

  web:
    image: ghcr.io/nlbi/nlbi:latest
    restart: always
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://nlbi:@db:5432/nlbi
      NEXTAUTH_SECRET_FILE: /run/secrets/nextauth_secret
      NODE_ENV: production
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 1G
    secrets:
      - nextauth_secret
      - db_password

  pgbouncer:
    image: bitnami/pgbouncer:latest
    ports:
      - "6432:6432"
    environment:
      POSTGRESQL_HOST: db
      POSTGRESQL_DATABASE: nlbi
      PGBOUNCER_POOL_MODE: transaction
      PGBOUNCER_MAX_CLIENT_CONN: 200
      PGBOUNCER_DEFAULT_POOL_SIZE: 20
    depends_on:
      - db

volumes:
  pgdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt
  nextauth_secret:
    file: ./secrets/nextauth_secret.txt
```

**Testing:**
- Docker Compose production stack starts cleanly and serves requests
- Query result caching: second identical query is served from cache (0ms execution time)
- OpenTelemetry traces appear in Jaeger/Grafana Tempo for each query pipeline execution
- Prometheus metrics endpoint exposes query_latency_seconds, llm_tokens_total, cache_hit_ratio
- CI pipeline passes: lint, type check, 90%+ unit test coverage, integration tests against test PostgreSQL
- Helm chart deploys to a Kubernetes cluster and passes readiness/liveness probes
- Self-hosting guide: a new user can go from zero to running instance in under 15 minutes following the documentation
- Load test: system handles 50 concurrent users with p95 query latency < 5 seconds (excluding LLM call time)

**Estimated duration:** 3 weeks

---

## Dependency Graph

```
Phase 1: Foundation
  │
  ├──► Phase 2: Data Source Connectors & Schema Sync
  │       │
  │       ├──► Phase 3: Semantic Layer Integration (Cube.js)
  │       │       │
  │       │       └──► Phase 4: NL-to-SQL Query Engine ◄── (depends on Phase 2 + Phase 3)
  │       │               │
  │       │               ├──► Phase 5: Visualization Engine
  │       │               │       │
  │       │               │       └──► Phase 6: Dashboards & Sharing
  │       │               │               │
  │       │               │               ├──► Phase 9: Alerts & Scheduled Reports
  │       │               │               │
  │       │               │               └──► Phase 10: Schema Change Detection
  │       │               │
  │       │               ├──► Phase 7: Automated Insight Narration
  │       │               │
  │       │               └──► Phase 8: Multi-Turn Conversation & Decomposition
  │       │
  │       └──► Phase 10: Schema Change Detection ◄── (also depends on Phase 6)
  │
  └──► Phase 11: Embeddable Widget SDK ◄── (depends on Phase 4 + Phase 5)
          │
          └──► Phase 12: Production Hardening ◄── (depends on all prior phases)
```

**Critical path:** Phase 1 -> Phase 2 -> Phase 3 -> Phase 4 -> Phase 5 -> Phase 6 -> Phase 12

**Parallelizable after Phase 4:**
- Phase 7 (Insight Narration) can proceed in parallel with Phase 5/6
- Phase 8 (Multi-Turn Conversation) can proceed in parallel with Phase 5/6
- Phase 11 (Embed SDK) can proceed once Phase 4 + Phase 5 are complete

---

## Definition of Done

A phase is considered "done" when all of the following criteria are met:

### Code Quality
- [ ] All TypeScript code compiles with `strict: true` and zero type errors
- [ ] ESLint passes with zero errors (warnings acceptable for intentional suppressions)
- [ ] Unit test coverage for new code is >= 85% (measured by Vitest + c8)
- [ ] Integration tests pass against a real PostgreSQL 16 + pgvector instance

### Functionality
- [ ] All features listed in the phase's "What" section are implemented and functional
- [ ] All test scenarios listed in the phase's "Testing" section pass
- [ ] No regressions: all tests from prior phases continue to pass

### Documentation
- [ ] New API endpoints have OpenAPI annotations
- [ ] New database tables have migration files with rollback support
- [ ] Complex algorithms have inline code comments explaining the approach

### Deployment
- [ ] Docker Compose development stack starts and serves the new features
- [ ] Database migrations apply cleanly on a fresh database and on an existing database from the previous phase
- [ ] Environment variables for new features are documented in `.env.example`

### Review
- [ ] Code reviewed by at least one other contributor
- [ ] No P0/P1 bugs in the feature (P2 bugs may be deferred with a tracking issue)

---

## Timeline Summary

| Phase | Name | Duration | Cumulative |
|-------|------|----------|------------|
| 1 | Foundation | 2 weeks | Week 2 |
| 2 | Data Source Connectors & Schema Sync | 3 weeks | Week 5 |
| 3 | Semantic Layer Integration | 2 weeks | Week 7 |
| 4 | NL-to-SQL Query Engine | 3 weeks | Week 10 |
| 5 | Visualization Engine | 2 weeks | Week 12 |
| 6 | Dashboards & Sharing | 3 weeks | Week 15 |
| 7 | Automated Insight Narration | 2 weeks | Week 17 |
| 8 | Multi-Turn Conversation & Decomposition | 3 weeks | Week 20 |
| 9 | Alerts & Scheduled Reports | 2 weeks | Week 22 |
| 10 | Schema Change Detection | 2 weeks | Week 24 |
| 11 | Embeddable Widget SDK & API | 3 weeks | Week 27 |
| 12 | Production Hardening | 3 weeks | Week 30 |

**MVP (Phases 1-6):** 15 weeks / ~3.5 months
**Full v1.0 (Phases 1-12):** 30 weeks / ~7.5 months

Phases 7, 8, and 11 can be parallelized with Phases 5-6 if a second developer is available, reducing the full timeline to approximately 22-24 weeks / ~5.5 months with two developers.
