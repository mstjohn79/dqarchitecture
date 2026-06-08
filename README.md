# Data Quality Architecture — Crawl, Walk, Run

A progressive framework for implementing enterprise data quality on Snowflake using native capabilities. Built by Evolv Solutions Engineering.

---

## What's In This Repo

| Document | Description |
|----------|-------------|
| [**Crawl, Walk, Run Framework**](crawl-walk-run-framework.md) | The full maturity framework — 57 assets across 3 tiers, from zero-setup ad-hoc checks to automated pipeline protection |
| [**Snowflake-Native DQ Architecture**](snowflake_native_dq_architecture.md) | Technical architecture replacing LangChain/CrewAI with Cortex Agents, DMFs, and native Snowflake tooling |
| [**DQ Dashboard Guide (Vertiv Example)**](vertiv_dq_dashboard_guide.md) | Reusable Streamlit dashboard pattern with domain scoring, trend analysis, and AI-driven recommendations |

---

## The Three Tiers

```
┌─────────────────────────────────────────────────────────────────────┐
│  RUN: Automated Governance                                          │
│  SLA alerts, circuit breakers, incident investigation,              │
│  meta-observability, cross-system orchestration                     │
├─────────────────────────────────────────────────────────────────────┤
│  WALK: Continuous Monitoring                                        │
│  DMFs deployed, health scores, trends, regression detection,        │
│  custom rules, expectations management                              │
├─────────────────────────────────────────────────────────────────────┤
│  CRAWL: Foundation & Awareness                                      │
│  Ad-hoc assessments, table comparisons, usage analytics             │
│  Zero setup required                                                │
└─────────────────────────────────────────────────────────────────────┘
```

### Crawl — "Can I trust my data right now?"
- One-time quality scans (no DMFs needed)
- Table comparison & migration validation
- Dataset popularity & usage analytics
- **Zero configuration required**

### Walk — "Is quality getting better or worse?"
- Automated DMF deployment recommendations
- Schema-wide health scoring
- Root cause analysis & regression detection
- 30-day trend analysis
- Custom business rules (format, range, FK integrity)

### Run — "Bad data never reaches production"
- Automated SLA alerts on quality drops
- Circuit breakers that halt pipelines on violations
- Coverage gap & cost optimization analysis
- Multi-skill incident investigation (DQ + lineage + governance)

---

## Snowflake Features Used

- **Data Metric Functions** (system + custom)
- **Cortex Agents** for orchestration
- **Cortex AI Functions** for intelligent profiling
- **Snowflake ALERTs** for SLA monitoring
- **Snowflake TASKs** for pipeline control (circuit breakers)
- **Streamlit in Snowflake** for dashboards
- **ACCOUNT_USAGE** for popularity & cost analysis
- **Cortex Complete** for prompt quality scoring

---

## How to Use This With a Client

1. **Start at Crawl** — Run an ad-hoc assessment on their most critical schema. Show quality today with zero setup.
2. **Propose Walk** — Based on findings, recommend DMF deployment. Show the health scoring and trends they'll get.
3. **Graduate to Run** — Layer in SLA alerts, circuit breakers, and coverage analysis for full automated governance.

Each tier builds on the previous — nothing is throwaway work.

---

## Total Collateral

| Category | Count |
|----------|-------|
| Workflows | 19 |
| SQL Templates | 31 |
| Reference Docs | 5 |
| Architecture Docs | 2 |
| Dashboard Guide | 1 |
| **Total** | **58 assets** |
