# Natural Language BI Platform

**Status:** Candidate Project  
**Market Size:** $34.8B (2025) → $72.2B (2034) at 8.4% CAGR (BI market); Augmented Analytics growing at 17.4% CAGR  
**Last Updated:** 2026-05-02

## Overview

An open-source, self-hostable natural language business intelligence platform that treats semantic accuracy as a first-class citizen. Today's NL BI tools (ThoughtSpot, Power BI Copilot, Tableau Einstein) struggle with complex schemas and ambiguous metrics. This project solves accuracy through:

- **Mandatory semantic layer grounding:** Accuracy improves from 70–85% baseline to 90%+ through Cube.js or dbt Semantic Layer integration
- **Conversational drill-down with context retention:** "Now break that down by region"—without re-specifying the full query
- **Automated insight narration:** LLM-generated written narratives alongside charts explaining what changed and why it's notable
- **Self-healing dashboards:** Auto-detect schema changes and propose semantic-similarity-based column mappings
- **Open-source + permissive license:** Escape vendor lock-in; modify and white-label freely

## The Market Gap

The BI market is bifurcated:

1. **Enterprise vendors ($25–$200/user/month):** ThoughtSpot, Power BI Copilot, Tableau Einstein—mature but expensive and vendor-locked
2. **Open-source BI (free):** Metabase, Superset—great for dashboards but zero NL querying out of the box
3. **Emerging AI-native SaaS ($20–$5,000/mo):** Julius AI, camelAI, Supaboard—no semantic layer, high hallucination rates, expensive at scale

**The gap:** No open-source tool combines production-grade NL querying with semantic grounding. The Cube.js + Metabase combination is technically possible but requires substantial engineering. Teams want:
- Free or low-cost entry (eliminates $100K+/yr contracts)
- NL query accuracy on real enterprise schemas (hallucination is a blocker)
- Self-hosting for data sovereignty
- Permissive licensing for white-labeling

## Core Features

### MVP (Must-Have)
- Natural language question interface producing a chart or table result, grounded in a semantic layer (Cube.js or dbt Semantic Layer) to anchor accuracy and prevent hallucination
- Connectors for at least PostgreSQL, MySQL, BigQuery, and Snowflake as initial data sources
- Multi-turn conversational context: follow-up questions that reference and refine the previous result without re-specifying the full query
- Role-based access control ensuring users can only query data within their permission scope
- Dashboard creation and shareable link generation from NL-derived visualisations

### Should-Have (v1.1)
- **Automated insight narration:** LLM-generated written summary of what the data shows, what changed, and what is notable—alongside each visualisation
- **Schema-aware query accuracy improvements:** RAG over schema documentation and metric definitions; query decomposition for multi-join questions
- Anomaly and trend alerts: scheduled monitoring of key metrics with NL narrative alerts delivered via Slack or email
- Embeddable NL query widget with signed-URL embedding API for SaaS product integration

### Nice-to-Have (Backlog)
- **Self-healing dashboards:** Detect broken column/table references after schema migrations and propose semantic-similarity-based fixes
- Proactive insight delivery: push weekly metric summaries to users without requiring them to log in
- NL-driven semantic model authoring: allow data engineers to describe a metric in plain language and have the system generate Cube.js or dbt YAML definition
- Export to presentation formats (PDF, slides) with LLM-generated narrative captions per slide

## AI-Native Opportunities

1. **Accuracy through semantic grounding**
   - Current NL-to-SQL tools achieve 70–85% accuracy on clean-schema benchmarks; real enterprise schemas are messier
   - A platform with mandatory semantic layer integration (dbt Semantic Layer / Cube) can push accuracy to 90%+ by constraining LLM output to governed metrics only
   - This closes the gap that makes current tools untrustworthy for business decisions

2. **Conversational drill-down and context retention**
   - Existing tools treat each query as stateless
   - An AI-native platform maintaining session context enables: "now break that down by region," "exclude the Q2 outlier," "compare to same period last year"—reducing time-to-insight from hours to seconds

3. **Automated insight narration**
   - Most BI platforms produce charts that users must interpret
   - An LLM layer generates written narratives alongside visualizations—explaining what changed, why it's notable (anomaly/trend), and what action it implies—making dashboards accessible to non-analysts

4. **Self-healing and schema-aware query repair**
   - When schema changes break dashboards (renamed columns, deprecated tables), current systems fail silently or require manual fixes
   - An AI agent could detect failures, map old references to new ones using semantic similarity, and propose or auto-apply fixes

5. **Open-source differentiation**
   - Power BI Copilot, Tableau Einstein, ThoughtSpot are tightly coupled to vendor ecosystems and pricing
   - An open-source platform with pluggable connectors (any SQL source), permissive license, and community-governed semantic layer would be the first true open-source NL BI alternative

## Competitive Landscape

| Tool | Type | Semantic Layer | Accuracy | Cost | Self-Hosted |
|------|------|---|----------|------|-------------|
| **ThoughtSpot (Spotter)** | Commercial SaaS | ✓ (Spotter Semantics) | 85%+ | $25–$300K/yr | Limited |
| **Power BI Copilot** | Commercial SaaS | Limited | 75% | $14/user/mo + $262/mo Fabric | ❌ |
| **Tableau Einstein** | Commercial SaaS | Partial | 80% | $75+/user/mo + premium | ❌ |
| **Looker (Gemini)** | Commercial SaaS | ✓ (LookML) | 85%+ | $30K–$150K/yr | Limited |
| **Metabase** | OSS + SaaS | ❌ | 40% (without Semantic Layer) | Free self-hosted | ✓ |
| **Cube.js** | OSS semantic layer | ✓ | N/A (engine-agnostic) | Free self-hosted | ✓ |
| **This Project** | OSS + SaaS | ✓ (required) | 90%+ | Free self-hosted | ✓ |

## Technical Design Considerations

- **Architecture:** Semantic layer integration required; support Cube.js (REST/GraphQL/SQL APIs) and dbt Semantic Layer (MetricFlow)
- **NL to SQL:** LLM (Claude, OpenAI) with RAG over semantic model definitions; query decomposition for multi-join questions (DIN-SQL approach)
- **Session state:** In-memory context management per user; optional Redis for distributed deployments
- **Visualization:** Embed or build on Apache Superset's 40+ chart types
- **Accuracy grounding:** Semantic layer constraints prevent hallucination; LLM output must conform to defined metrics
- **Deployment:** Docker Compose or Kubernetes; managed SaaS option with team collaboration

## Market Validation

- **Market drivers:**
  - Augmented analytics (AI-powered BI) growing at 17.4% CAGR—fastest-growing sub-segment
  - Citizen analyst demand: business users need ad-hoc querying without data team dependency
  - Vendor lock-in fatigue: enterprises evaluating open-source alternatives to Power BI/Tableau/Looker

- **Customer personas:**
  - Business analysts at mid-to-large enterprises without dedicated SQL skills
  - Data-forward SMBs (50–500 employees) that cannot afford $100K/yr contracts
  - Product managers and C-suite executives wanting ad-hoc query access
  - Embedded analytics buyers (SaaS companies white-labeling NL BI)

## Why Build This

1. **Market timing:** Augmented analytics growing at 17.4% CAGR; open-source gap is wide
2. **Technology maturity:** Text-to-SQL via LLMs is well-researched; semantic layer standards (Cube, dbt) are established
3. **Commercial opportunity:** Build on free tier; offer managed SaaS for enterprises (similar to Metabase model)
4. **Platform leverage:** Cube.js provides semantic layer; Superset provides UI; Claude API adds conversational layer

## Success Metrics

- **Adoption:** 1K+ GitHub stars within 12 months; featured as Metabase alternative for NL querying
- **Accuracy:** Achieve 90%+ query accuracy on real enterprise schemas (vs. 70–85% baseline)
- **Commercial:** Win 10+ SMB/mid-market customers with managed SaaS tier
- **Enterprise:** Demonstrate ROI (reduce time-to-insight from hours to minutes) with 5+ case studies

---

**Resources:**
- [Cube.js Semantic Layer Documentation](https://cube.dev/docs/product/data-modeling/overview)
- [dbt Semantic Layer (MetricFlow)](https://docs.getdbt.com/docs/use-cases/metrics/overview)
- [ThoughtSpot Spotter Documentation](https://cloud-docs.thoughtspot.com/en/latest/)
- [Text-to-SQL with LLMs Survey](https://arxiv.org/abs/2308.15363)
- [Apache Superset Documentation](https://superset.apache.org/docs/)
