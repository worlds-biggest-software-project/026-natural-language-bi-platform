# Natural Language BI Platform — Feature & Functionality Survey

> Candidate #26 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| ThoughtSpot (Spotter) | Commercial SaaS | Proprietary / from $25/user/month | https://www.thoughtspot.com |
| Power BI Copilot | Commercial SaaS | Proprietary / Pro $14/user/month + Fabric capacity | https://powerbi.microsoft.com |
| Tableau (Einstein / Tableau Pulse) | Commercial SaaS | Proprietary / Creator $75/month + Tableau+ premium | https://www.tableau.com |
| Looker (Gemini integration) | Commercial SaaS | Proprietary / enterprise quote | https://cloud.google.com/looker |
| Metabase | Open source + commercial SaaS | AGPL-3.0 (OSS) / commercial cloud | https://www.metabase.com |
| Apache Superset | Open source | Apache 2.0 | https://superset.apache.org |
| Cube (Cube.js) | Open source semantic layer + commercial | Apache 2.0 (OSS) / Cube Cloud from $399/month | https://cube.dev |
| Julius AI | Commercial SaaS | Proprietary / freemium; from ~$20/month | https://julius.ai |
| camelAI | Commercial SaaS | Proprietary / freemium | https://camelai.com |
| Supaboard | Commercial SaaS | Proprietary / from ~$49/month | https://supaboard.ai |
| Lightdash | Open source | MIT (OSS) / commercial cloud | https://www.lightdash.com |

---

## Feature Analysis by Solution

### ThoughtSpot (Spotter)

**Core features**
- Search-first NL query model: users type or speak natural language questions; the platform generates and executes the underlying query
- Spotter AI agent: multi-step reasoning engine that breaks down complex questions, tests assumptions, cross-checks results, and delivers recommended actions
- Spotter Semantics (GA March 2026): agentic semantic layer that transforms raw data into governed business context, ensuring every NL query returns an accurate and explainable answer
- SpotterViz agent (2026): builds dashboards from natural language descriptions
- SpotterModel agent (2026): builds semantic models without writing code
- SpotterCode agent (2026): generates embedded analytics application code via AI-assisted code generation
- Analyst Studio: blends self-service BI with data modelling (GA early 2025, evolved from Mode acquisition)
- Spotter 3 integration of unstructured data sources: pulls context from Jira, Slack, Salesforce, and SharePoint alongside structured databases

**Differentiating features**
- Only mature BI platform with an agentic architecture where multiple specialised AI agents handle distinct workflow stages (querying, visualisation, modelling, embedding)
- Spotter Semantics is the most advanced production semantic governance layer tied to an NL query engine in the commercial market as of March 2026
- 64% of all ThoughtSpot customers actively use Spotter as their primary analyst interface (platform usage data, FY2025)

**UX patterns**
- Search bar as the primary entry point; designed for non-technical users from day one
- Auto-generated pinboards and dashboards from search results
- Conversational follow-up within a session ("now break that down by region")
- Proactive AI-generated recommendations alongside query results

**Integration points**
- Connects to Snowflake, BigQuery, Redshift, Databricks, and major cloud data warehouses
- Embedding SDK for integrating Spotter into third-party SaaS products
- REST API and ThoughtSpot Everywhere for embedded analytics
- Salesforce, Slack, Microsoft Teams for in-workflow analytics delivery

**Known gaps**
- Price barrier: entry pricing appears accessible but enterprise contracts average $100K–$300K/yr
- Vendor lock-in: semantic models and content are proprietary; migration is non-trivial
- Complex multi-join queries and ambiguous schema still produce unreliable results without careful semantic model setup
- Spotter agents (SpotterViz, SpotterModel, SpotterCode) were scheduled for GA in early 2026; production maturity at the time of research is still being established

**Licence / IP notes**
- Fully proprietary commercial SaaS. Total funding approximately $801M. No open-source components. Patent portfolio expected on NL-to-search techniques; specific patent numbers not confirmed in public sources.

---

### Power BI Copilot

**Core features**
- Natural language report and visual generation: users describe a visual in plain English and Copilot generates it within an existing report
- DAX formula generation from natural language: interprets user intent and produces correct DAX syntax
- Semantic model context attachment: Copilot reads user-specific report schemas and custom field relationships to improve query accuracy
- Verified Answers: pre-mapped natural language patterns to specific metrics with filter application; substantially improved in 2025 updates
- Copilot chat pane in Power BI Service for conversational exploration of semantic models
- Legacy Power BI Q&A being deprecated December 2026; Copilot replaces it entirely

**Differentiating features**
- Deepest Microsoft 365 ecosystem integration: Copilot in Power BI shares the same Copilot identity as Word, Excel, and Teams, enabling cross-product data surfacing
- Predictive modelling and automated recommendations roadmap for 2026: Power BI Copilot is evolving beyond query answering toward autonomous insight generation
- Fabric capacity enables unified analytics across lakehouses, warehouses, and real-time streams with a shared Copilot interface

**UX patterns**
- Copilot pane within Power BI Desktop and Service; side-by-side with existing reports
- Natural language prompts generate visuals inline in the report canvas
- Consistent cross-platform experience: Desktop, web, and mobile from 2025 onwards

**Integration points**
- Native to Microsoft Fabric ecosystem; deep integration with Azure Data Factory, Synapse, OneLake
- Power BI REST API for embedding and programmatic access
- Microsoft Teams and SharePoint for collaborative report sharing
- Copilot requires Fabric F2 capacity (~$262/month minimum) in addition to per-user Pro licences

**Known gaps**
- Copilot features are gated behind Fabric capacity licences, making the effective entry cost significantly higher than the nominal $14/user/month Pro price
- No production-grade open API for third-party NL integration; tightly coupled to Microsoft data ecosystem
- Complex cross-database joins and non-Microsoft data sources have reduced Copilot accuracy compared to native Fabric sources
- Legacy Q&A deprecation (December 2026) creates migration effort for existing customers
- Privacy controls limit what data Copilot can access in sovereign cloud and GCC deployments

**Licence / IP notes**
- Fully proprietary Microsoft product. Governed by Microsoft Services Agreement and Microsoft Product Terms. No open-source components. Extensive patent portfolio on AI and BI techniques.

---

### Tableau (Einstein / Tableau Pulse / Tableau Agent)

**Core features**
- Tableau Pulse: delivers natural-language metric summaries, proactive alerts, and automated insight narratives embedded in workflow via email, Slack, and mobile
- Tableau Agent: generative AI and NL query interface guiding users through analytics; describes transformations in plain language
- Explain Data: automated statistical explanation of data points and anomalies surfaced alongside visualisations
- Data Pro (launched Tableau 2025.2, August 2025): AI semantic modelling assistant powered by Agentforce; automates construction of semantic models
- Tableau Semantics and Tableau Marketplace (AI features, GA scheduled 2025): governed metric layer for consistent AI query accuracy
- All AI capabilities built on the Agentforce Trust Layer (formerly Einstein Trust Layer) for data security and ethical AI governance

**Differentiating features**
- Tableau Pulse's proactive push model uniquely delivers insight narratives to non-analyst users in their existing workflow (email, Slack, mobile) without requiring them to open a BI tool
- Agentforce Trust Layer provides enterprise-grade governance, audit trails, and privacy controls over AI-generated analytics content
- Deepest Salesforce CRM integration: natural language queries can span CRM pipeline data and product/financial data in a single question

**UX patterns**
- Traditional Tableau drag-and-drop authoring supplemented by NL prompts and AI suggestions
- Tableau Pulse is consumption-first: metrics surface as cards with natural-language captions; users do not need to build views
- Tableau Agent guides non-technical users through analytical steps conversationally

**Integration points**
- Native Salesforce, Tableau Cloud, Snowflake, Databricks, BigQuery, and Redshift connectors
- Tableau Prep for data pipeline transformation
- REST API and JavaScript embedding API (Tableau Embedded Analytics)
- Slack and Microsoft Teams for Pulse notification delivery

**Known gaps**
- Full NL/AI features require Tableau+ bundle at significant premium over base Creator/Explorer pricing; actual cost is sales-negotiated and opaque
- Tableau Einstein/Tableau+ are relatively new (2024–2025); maturity and reliability of AI features in enterprise deployments is still being established
- Salesforce platform coupling means Tableau is most valuable in Salesforce-centric organisations; cross-ecosystem deployments face complexity
- No open-source components; migration away from Tableau involves significant re-work

**Licence / IP notes**
- Fully proprietary; Salesforce acquisition ($15.7B, 2019). Extensive Salesforce and Tableau patent portfolio. No open-source components.

---

### Looker (Gemini integration)

**Core features**
- Conversational Analytics (GA as of 2025/2026): natural language queries translated into LookML-governed metric definitions; Looker's semantic layer reduces AI query errors by up to two-thirds in internal testing
- LookML Assistant: AI agent that generates LookML code from natural language business descriptions; VS Code and agentic IDE plugin available
- Visualization Assistant: natural language chart creation and refinement ("change to a stacked bar chart, colour-code by region")
- Code Interpreter: handles complex analytical tasks — forecasting, anomaly detection — by translating queries into Python code
- Agentic BI roadmap presented at Google Next '26: Looker positioning as an orchestration layer for analytical agents across the Google Cloud ecosystem
- LookML semantic model remains the anchor for all NL query accuracy: every answer is grounded in governed, business-defined metrics

**Differentiating features**
- The LookML semantic layer is the most mature and rigorously governed semantic model in the commercial BI market; grounding all NL queries in LookML reduces hallucination rates that afflict less-governed NL BI tools
- Gemini in Looker offers multi-modal analytical capability: Code Interpreter bridges natural language questions and programmatic analysis (Python, forecasting) in one flow
- LookML Assistant reduces the time-to-first-semantic-model for new deployments significantly, lowering the traditionally high Looker implementation cost

**UX patterns**
- Looker Explore: guided data discovery interface; Gemini Conversational Analytics adds NL entry point to existing Explore workflows
- LookML Developer experience: YAML/LiquidML semantic modelling; LookML Assistant makes this accessible to non-engineers
- Dashboard-first consumption: Looker dashboards with Gemini-generated chart variations and NL insight captions

**Integration points**
- Native Google Cloud: BigQuery, AlloyDB, Cloud Spanner; strong connectors for Snowflake, Redshift, Databricks
- Google Workspace (Sheets, Slides, Docs) via Looker Connected Sheets and Gemini
- Looker API and Looker SDK for embedding analytics in third-party applications
- Vertex AI for custom model integration within LookML workflows

**Known gaps**
- Enterprise quote-based pricing ($30K–$150K/yr) with no transparent self-serve tier
- LookML expertise remains a significant onboarding barrier; LookML Assistant reduces but does not eliminate the need for a skilled LookML developer
- Tightly coupled to Google Cloud; organisations on Azure or AWS face higher integration complexity
- Conversational Analytics, while accurate for LookML-defined metrics, cannot answer questions outside the semantic model's governed scope

**Licence / IP notes**
- Fully proprietary; Google acquisition ($2.6B, 2020). No open-source components. Extensive Google AI patent portfolio. LookML language specification is proprietary.

---

### Metabase

**Core features**
- Visual notebook-style query builder: non-technical users construct queries via GUI without writing SQL
- Metabot AI assistant (current offering): natural language questions, SQL generation and editing, AI-generated chart summaries, content search, and documentation Q&A
- Two-step agentic NL-to-SQL process (introduced February 2025): QueryDesigner Agent interprets natural language using the database schema; QueryArchitect Agent generates accurate SQL using table documentation
- Fast setup: single JAR file or Docker container; connects to PostgreSQL, MySQL, BigQuery, Snowflake, Redshift, and others
- Dashboards, automated reports via subscriptions (email/Slack), and alerts

**Differentiating features**
- Widest self-hosting adoption in the open-source BI category; strong community and plugin ecosystem
- AGPL-3.0 licence with a vibrant community of third-party integrations and a rich hosted cloud offering
- Lowest time-to-first-dashboard of any tool reviewed: non-technical users can connect a database and produce a chart in under 30 minutes

**UX patterns**
- "Ask a question" as the primary entry point (GUI-based or NL via Metabot)
- Browse-first interface: users can explore existing questions, dashboards, and collections without authoring
- Question notebook: step-by-step visual query construction with optional NL shortcut via Metabot

**Integration points**
- 20+ native database connectors (PostgreSQL, MySQL, SQLite, Redshift, BigQuery, Snowflake, Databricks, MongoDB, etc.)
- Slack and email for automated report delivery
- REST API and embedding SDK (iFrame + signed embedding for whitelabelled deployments)
- MCP server (community-built, 2025): integrates Metabase directly with Claude, ChatGPT, and Cursor for conversational analytics via LLM tools

**Known gaps**
- Native NL querying (Metabot) is not available in the open-source community edition; it requires a paid cloud or enterprise licence
- Metabot's NL accuracy degrades significantly on complex multi-join schemas without a formal semantic model layer; no equivalent of LookML or dbt Semantic Layer to anchor queries
- No built-in semantic layer; metric consistency across teams relies on saved questions and sandboxing, which do not scale to large organisations
- Limited support for very large datasets; not designed for petabyte-scale warehouses
- AGPL-3.0 licence requires open-sourcing modifications, which constrains proprietary SaaS deployments that embed Metabase

**Licence / IP notes**
- Open-source community edition: AGPL-3.0. The AGPL copyleft obligation applies to modifications of Metabase itself. SaaS products embedding modified Metabase must open-source their changes unless they use the commercial Enterprise licence. No patent concerns identified in public sources.

---

### Apache Superset

**Core features**
- Rich SQL-based chart builder with 40+ visualisation types; designed for SQL-proficient teams
- SQL Lab: full-featured SQL IDE within the browser with query history, schema browser, and table preview
- Role-based access control (RBAC) and row-level security for enterprise data governance
- Can be paired with Cube.js or dbt Semantic Layer for governed metric definitions
- No native NL-to-chart capability as of April 2026; external LLM API integration is possible via community plugins
- Supports Snowflake, BigQuery, Redshift, PostgreSQL, MySQL, ClickHouse, Druid, Trino, and others via SQLAlchemy

**Differentiating features**
- Most technically capable and extensible open-source BI tool available; preferred by data-engineering-heavy teams
- Apache 2.0 licence: maximally permissive for commercial use and modification with no copyleft obligations
- Preset (commercial host of Superset) offers a managed AI-native layer on top of open Superset — the nearest commercial analogue to a hosted open-source NL BI product

**UX patterns**
- Developer and analyst-centric; assumes SQL familiarity
- Chart builder is visual but requires understanding of dimensions, measures, and SQL concepts
- No conversational or NL entry point in the base product

**Integration points**
- SQLAlchemy-based connectors for virtually any SQL database
- REST API and embedded SDK
- Slack notifications for dashboards
- Integrates with Apache Airflow for pipeline-triggered dashboard refreshes

**Known gaps**
- No native NL querying; requires external LLM integration effort to add
- Installation and maintenance are complex; recommended Kubernetes or Docker Compose deployment requires DevOps expertise
- RBAC configuration is non-trivial for large multi-tenant deployments
- Community support is broad but fragmented; enterprise support requires Preset or similar managed service

**Licence / IP notes**
- Apache 2.0: maximally permissive. No copyleft; no patent concerns identified. Any project building on Superset can do so freely with no source-disclosure obligations.

---

### Cube (Cube.js)

**Core features**
- Universal semantic layer: exposes metrics, dimensions, and their relationships via REST, GraphQL, SQL, MDX, and AI APIs across any SQL data source
- AI API: turnkey interface for connecting LLMs (including Anthropic Claude) to the semantic layer for NL-to-query workflows with built-in context injection
- Cube D3 (launched June 2025): adds two specialised AI agents — AI Data Analyst (NL querying with visualisations) and AI Data Engineer (automated semantic model development from cloud data sources)
- Analytics Chat (launched October 2025): user-facing conversational interface for non-technical users to query curated semantic models
- Multi-tenancy, caching (pre-aggregations), and query acceleration built into the data access layer
- Named GigaOm 2025 Radar Leader with Outperformer status for innovation pace

**Differentiating features**
- The only open-source semantic layer with a production-ready AI API that any LLM can call — making it the architectural foundation for building a governed NL BI tool from scratch
- Code-first semantic model: defined in YAML/JavaScript, version-controlled, and deployable via CI/CD — unique among semantic layer tools
- Vendor-neutral: works with any SQL warehouse (Snowflake, BigQuery, Redshift, Databricks, PostgreSQL, ClickHouse, etc.)

**UX patterns**
- Developer and data-engineering focused: primary authoring is code (YAML/JavaScript data models)
- Cube Playground: web UI for exploring the semantic model and testing queries
- Analytics Chat provides a non-technical user interface but is a Cube Cloud feature

**Integration points**
- REST, GraphQL, SQL, and MDX APIs for BI tool consumption (Superset, Metabase, Redash, Grafana, Tableau, Power BI, Looker Studio)
- AI API for LLM tool integration (Claude, OpenAI GPT, etc.)
- dbt integration: can consume dbt models and metrics as the semantic foundation
- Cube Cloud for managed hosting with enterprise features (access control, SCIM, audit logs)

**Known gaps**
- Not a user-facing BI front-end; requires a separate visualisation layer
- Analytics Chat and AI Data Engineer/Analyst agents are Cube Cloud (commercial) features, not available in the open-source core
- Cold query performance (non-pre-aggregated paths) requires careful tuning for large datasets
- Community edition lacks multi-tenant isolation; enterprise multi-tenancy requires Cube Cloud

**Licence / IP notes**
- Core Cube.js: Apache 2.0 — maximally permissive. Cube Cloud and AI features: proprietary commercial. No patent encumbrance identified on the open-source core. Building an NL BI platform on the Cube.js semantic layer introduces no copyleft obligations.

---

### Julius AI / camelAI / Supaboard

**Core features**
- Julius AI: file-upload-centric NL analytics (CSV, Excel, Google Sheets); statistical analysis and chart generation via conversational prompts; basic database connectors (architecture remains file-centric)
- camelAI: live database connection-first (PostgreSQL, Snowflake, BigQuery, MySQL, ClickHouse, and 40+ sources); NL-to-SQL with team collaboration; positions as the strongest database-native Julius alternative
- Supaboard: one-click integration with 60+ data sources including MySQL, PostgreSQL, Google Analytics, Shopify, Salesforce, and Notion; NL dashboard generation for non-technical SMB users; no-code emphasis

**Differentiating features**
- Fastest time-to-first-insight for individual analysts and small teams at $20–$49/month
- camelAI's live database connections avoid file-upload friction present in Julius
- Supaboard's breadth of SaaS connectors (Shopify, Salesforce, Google Analytics) targets non-data-engineering SMB teams

**UX patterns**
- Chat interface as primary interaction model; no GUI query builder
- Charts generated inline in the chat thread
- Minimal onboarding: connect data source, ask questions

**Integration points**
- Database connectors: varies by tool (Julius: limited; camelAI: 40+; Supaboard: 60+ including SaaS sources)
- No embedding SDK; not designed for embedded analytics in third-party products
- CSV/Excel/Google Sheets upload as universal fallback

**Known gaps**
- No semantic layer or governed metric definitions; NL accuracy degrades on complex, large, or poorly documented schemas
- Not suitable for enterprise multi-tenant or regulated environments
- No version control, CI/CD integration, or audit trails for queries
- Chart quality and customisation limited compared to mature BI tools
- camelAI and Supaboard are early-stage SaaS products; long-term viability and feature maturity are unverified

**Licence / IP notes**
- All three are fully proprietary commercial SaaS. No open-source components. No patent concerns identified in public sources.

---

### Lightdash

**Core features**
- Open-source BI platform natively built on dbt: reads dbt YAML metric and dimension definitions as the semantic foundation; no duplication of business logic in a separate BI layer
- AI analyst co-pilot for NL queries over dbt-defined metrics
- Supports Snowflake, BigQuery, PostgreSQL, Redshift, Databricks, and Trino via dbt adapters
- Dashboards, charts, and self-serve data exploration for non-technical business users
- Positions as "Agentic BI" in its 2025–2026 roadmap: analytics at the speed of code

**Differentiating features**
- The only open-source BI tool with dbt-native semantic layer integration as a first-class design principle; eliminates the dual-modelling problem present in all other BI tools
- MIT licence: maximally permissive for commercial use and modification

**UX patterns**
- Developer setup: connects to an existing dbt project; business users consume pre-defined metrics and dashboards
- SQL-mode and visual chart builder for analyst exploration
- Low authoring friction for teams already on dbt

**Integration points**
- dbt Cloud and dbt Core (all major adapters)
- Slack notifications for scheduled dashboards and alerts
- REST API; embedding SDK in development

**Known gaps**
- Requires an existing dbt project; not suitable for teams without a modern data stack
- NL/AI co-pilot is less mature than commercial offerings (Looker Gemini, ThoughtSpot Spotter)
- Smaller community and ecosystem than Metabase or Superset
- Cloud-hosted version with full AI features is commercial; self-hosted open-source version has reduced AI feature availability

**Licence / IP notes**
- Open-source: MIT licence. Maximally permissive; no copyleft or patent concerns identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Natural language question input producing a chart or table result without requiring SQL knowledge
- Connection to at least the major SQL data warehouses (PostgreSQL, MySQL, BigQuery, Snowflake, Redshift)
- Dashboard creation and sharing from NL-derived results
- Session continuity: the system understands follow-up questions in context ("now break that down by region")
- Result accuracy guardrails: structured output grounded in a schema or semantic model to prevent hallucination
- Role-based access control so users can only query data they are authorised to see

### Differentiating Features
- Semantic layer integration (dbt / Cube) as the accuracy anchor: governed metric definitions that LLM-generated SQL must conform to
- Conversational drill-down with full session memory across a multi-turn analytical dialogue
- Automated insight narration: written explanation of what changed, why it is notable, and what action it implies — delivered alongside visualisations
- Self-healing dashboards: AI detects broken references caused by schema changes and proposes or auto-applies fixes
- Proactive insight delivery: push summaries and anomaly alerts to Slack/email without requiring users to open the tool

### Underserved Areas / Opportunities
- Open-source NL BI with production-grade accuracy: no widely deployed open-source tool offers NL querying with chart generation out of the box; the Cube + Superset or Cube + Metabase combination requires significant integration effort
- Multi-turn conversational context retention: every current tool reviewed (commercial and open-source) treats NL queries as stateless by default
- NL accuracy on real enterprise schemas with ambiguous metric names and complex joins: frontier LLMs achieve 70–85% on clean benchmarks but degrade significantly on real-world schemas without a semantic grounding layer
- Embedded NL analytics for SaaS products: the embedded analytics market needs a permissively licensed, embeddable NL query component that SaaS companies can white-label

### AI-Augmentation Candidates
- Schema RAG: retrieval-augmented generation over schema documentation and metric definitions to ground LLM query generation in business context
- Automatic query decomposition for complex multi-join questions (DIN-SQL decomposition approach, NeurIPS 2023)
- Anomaly and trend detection with natural-language narrative explanation of root causes
- Schema change detection and automated dashboard repair when column names or table structures change
- Metric suggestion: proactively surfacing relevant metrics the user has not asked about but that are statistically related to their query

---

## Legal & IP Summary

All open-source tools in this survey have permissive or copyleft-with-exceptions licences. Apache Superset (Apache 2.0), Cube.js core (Apache 2.0), and Lightdash (MIT) are freely usable with no patent encumbrances identified in public sources and no source-disclosure obligations for downstream products. Metabase (AGPL-3.0) is copyleft: products that modify and distribute Metabase must open-source their changes, making it unsuitable as an embedded component in a proprietary SaaS product without a commercial licence. All commercial tools (ThoughtSpot, Power BI, Tableau, Looker) are fully proprietary with extensive patent portfolios in NL querying and semantic layer techniques; their interfaces, algorithms, and data models cannot be reproduced without licence. No specific patent encumbrances on the underlying NL-to-SQL techniques themselves (text-to-SQL via LLMs, retrieval-augmented generation, semantic layer query grounding) have been identified in public sources; these techniques derive from published academic research and are not known to be patent-blocked for new implementations.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Natural language question interface producing a chart or table result, grounded in a semantic layer (Cube.js or dbt Semantic Layer) to anchor accuracy and prevent hallucination
- Connectors for at least PostgreSQL, MySQL, BigQuery, and Snowflake as initial data sources
- Multi-turn conversational context: follow-up questions that reference and refine the previous result without re-specifying the full query
- Role-based access control ensuring users can only query data within their permission scope
- Dashboard creation and shareable link generation from NL-derived visualisations

**Should-have (v1.1)**
- Automated insight narration: LLM-generated written summary of what the data shows, what changed, and what is notable — alongside each visualisation
- Schema-aware query accuracy improvements: RAG over schema documentation and metric definitions; query decomposition for multi-join questions
- Anomaly and trend alerts: scheduled monitoring of key metrics with NL narrative alerts delivered via Slack or email
- Embeddable NL query widget with a signed-URL embedding API for SaaS product integration

**Nice-to-have (backlog)**
- Self-healing dashboards: detect broken column/table references after schema migrations and propose semantic-similarity-based fixes
- Proactive insight delivery: push weekly metric summaries to users without requiring them to log in
- NL-driven semantic model authoring: allow data engineers to describe a metric in plain language and have the system generate the Cube.js or dbt YAML definition
- Export to presentation formats (PDF, slides) with LLM-generated narrative captions per slide
