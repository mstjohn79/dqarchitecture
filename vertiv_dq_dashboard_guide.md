# Vertiv Data Quality Intelligence Dashboard

## Overview & Reuse Guide

This dashboard is the foundation of a **Crawl-Walk-Run** approach to enterprise data quality — starting with baseline visibility and building toward automated, AI-driven quality management, all running natively in Snowflake.

---

## The Data Model

Everything is driven by five core tables plus one materialized view, all in Snowflake:

| Table | Purpose |
|---|---|
| `DQ_DOMAINS` | Business domain registry (Customer & Sales Intelligence, Marketing Intelligence, Supply Chain & BOM Health, Sustainability & ESG), each with an assigned domain owner |
| `DQ_DATA_PRODUCTS` | Data products within each domain, linked to source tables and tagged with criticality levels (CRITICAL, HIGH, MEDIUM) |
| `DQ_SIGNAL_DEFINITIONS` | 36 DAMA-aligned quality signals, each with a scoring SQL expression, green/yellow/red thresholds, threshold direction, and a weight for composite scoring |
| `DQ_SIGNAL_RESULTS` | Measurement log — every signal run writes the signal value, row count, computed status, and timestamp here |
| `DQ_PRODUCT_HEALTH` | Rolled-up health scores per product per run, computed from weighted signal results |
| `DQ_KNOWN_GAPS` | Tracked data quality issues with severity, ownership, resolution status, and business impact |

A single view — **`VW_DQ_METRICS`** — joins the five core tables together and is what the dashboard queries. One view, four pages, no duplication.

### Source Tables

The six source tables represent the raw data being measured:

| Source Table | Rows | Domain |
|---|---|---|
| `SRC_ACCOUNTS` | 300 | Customer & Sales Intelligence |
| `SRC_CONTACTS` | 500 | Customer & Sales Intelligence |
| `SRC_INTENT_SIGNALS` | 400 | Customer & Sales Intelligence |
| `SRC_BOM_RISK_SCORES` | 500 | Supply Chain & BOM Health |
| `SRC_CAMPAIGN_TOUCHES` | 600 | Marketing Intelligence |
| `SRC_MQA_ACCOUNTS` | 350 | Marketing Intelligence |

To reuse this framework, swap these for whatever source tables you are profiling.

---

## The Approach — Crawl, Walk, Run

### Crawl (Current State)

Baseline visibility. SQL-computed signals across six DAMA data quality dimensions:

- **Accuracy** — Are values correct and within expected ranges?
- **Completeness** — Are required fields populated?
- **Consistency** — Do values agree across systems and tables?
- **Timeliness** — Is data current and refreshed within SLA?
- **Uniqueness** — Are keys and identifiers deduplicated?
- **Validity** — Do values conform to format and domain rules?

Each signal is scored against defined thresholds, rolled into product and domain health scores, and visualized in a Snowflake-hosted Streamlit dashboard.

### Walk (Next Phase)

Automated monitoring. Adds:

- **Snowflake Data Metric Functions (DMFs)** — both the four system DMFs (`NULL_COUNT`, `UNIQUE_COUNT`, `DUPLICATE_COUNT`, `FRESHNESS`) and custom DMFs — running on schedule with results attached directly to tables.
- **Cortex AI profiling** — `AI_CLASSIFY`, `AI_EXTRACT`, and `AI_COMPLETE` for semantic profiling: detecting PII, classifying column content, and generating plain-language quality summaries without moving data out of Snowflake.

### Run (Target State)

Closed-loop remediation:

- Quality violations trigger **Snowflake Tasks** and **alerts**
- **Cortex agents** orchestrate remediation workflows — root cause analysis, impact assessment, and recommended fixes
- The system shifts from "here's what's wrong" to "here's what we're doing about it"

---

## Architecture

The full architecture maps five agent roles from the original agentic DQ design to Snowflake-native services:

| Agent Role | Snowflake Implementation |
|---|---|
| **Profiling Agent** | DMFs + Cortex AI functions + metadata queries |
| **Monitoring Agent** | Scheduled Tasks + DMFs + alert conditions |
| **Assessment Agent** | Cortex `AI_COMPLETE` for scoring and root cause |
| **Remediation Agent** | Stored Procedures + Tasks for automated fixes |
| **Reporting Agent** | Streamlit dashboard + Cortex Analyst for natural language queries |

No external orchestrators. No data leaving Snowflake. Compute, storage, AI, and visualization all in one platform.

```
┌──────────────────────────────────────────────────────────────┐
│                    Snowflake Platform                         │
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │ Source    │──>│ Signal   │──>│ Product  │──>│ Streamlit│ │
│  │ Tables   │   │ Engine   │   │ Health   │   │Dashboard │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘ │
│       │              │              │               │        │
│       v              v              v               v        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │ DMFs     │   │ Cortex   │   │ Tasks &  │   │ Cortex   │ │
│  │(Walk)    │   │ AI (Walk)│   │Alerts    │   │ Analyst  │ │
│  │          │   │          │   │(Run)     │   │(Run)     │ │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## The Dashboard — Four Pages

### Overview

Leadership single-pane view:
- Overall health score across all products
- Domain-level status with red/yellow/green signal counts
- DAMA dimension breakdown as a stacked bar chart
- Product scorecards with composite health scores
- 30-day health score trend line

### Product Detail

Data steward drill-down:
- Select any data product to see individual signal scores against thresholds
- Signal metadata — DAMA dimension, weight, description
- 30-day signal-level mini-charts for trend analysis

### Known Gaps

Issue tracking:
- Gap summary KPIs (total, open, in progress, resolved, critical)
- Filters by status, domain, and DAMA dimension
- Gap cards with severity, ownership, business impact, and resolution notes
- Week-over-week change indicators

### Ownership & Actions

Accountability view:
- Domain ownership summary with average health scores
- Product ownership table with criticality and status
- Priority action items — any critical or high-severity product in yellow or red status gets flagged

---

## Reusing This for Your Own Build

### Step 1 — Create the Core Tables

Create the five core tables (`DQ_DOMAINS`, `DQ_DATA_PRODUCTS`, `DQ_SIGNAL_DEFINITIONS`, `DQ_SIGNAL_RESULTS`, `DQ_PRODUCT_HEALTH`) and the gap tracker (`DQ_KNOWN_GAPS`). DDL is in the repo.

### Step 2 — Register Your Domains and Products

Insert your business domains and the data products within them. Each product links to a source table and has a criticality level.

```sql
INSERT INTO DQ_DOMAINS (DOMAIN_NAME, DOMAIN_OWNER, DESCRIPTION)
VALUES ('Your Domain', 'Owner Name', 'Description');

INSERT INTO DQ_DATA_PRODUCTS (DOMAIN_ID, PRODUCT_NAME, PRODUCT_OWNER, CRITICALITY, SOURCE_TABLE)
VALUES (1, 'Your Product', 'Owner Name', 'HIGH', 'YOUR_SCHEMA.YOUR_TABLE');
```

### Step 3 — Write Signal Definitions

Each signal is a SQL expression that returns a numeric value, plus three thresholds. This is the only part you customize per use case.

```sql
INSERT INTO DQ_SIGNAL_DEFINITIONS (
    PRODUCT_ID, SIGNAL_NAME, DAMA_DIMENSION,
    SCORING_SQL, GREEN_THRESHOLD, YELLOW_THRESHOLD, RED_THRESHOLD,
    THRESHOLD_DIRECTION, WEIGHT
) VALUES (
    1, 'Email Completeness', 'Completeness',
    'SELECT ROUND(100.0 * COUNT(EMAIL) / COUNT(*), 2) FROM YOUR_TABLE',
    95, 80, 80, 'HIGHER_IS_BETTER', 2
);
```

### Step 4 — Run the Signals

Execute the scoring stored procedure. It evaluates all signal definitions, writes results to `DQ_SIGNAL_RESULTS`, and rolls up health scores to `DQ_PRODUCT_HEALTH`.

### Step 5 — Create the Metrics View

```sql
CREATE OR REPLACE VIEW VW_DQ_METRICS AS
SELECT
    sr.RUN_TS,
    DATE_TRUNC('day', sr.RUN_TS)::DATE AS RUN_DATE,
    dom.DOMAIN_NAME, dom.DOMAIN_OWNER,
    dp.PRODUCT_NAME, dp.PRODUCT_OWNER, dp.CRITICALITY,
    sr.SIGNAL_NAME, sr.DAMA_DIMENSION, sr.SIGNAL_VALUE,
    sr.ROW_COUNT, sr.STATUS,
    sd.GREEN_THRESHOLD, sd.YELLOW_THRESHOLD, sd.RED_THRESHOLD,
    sd.THRESHOLD_DIRECTION, sd.WEIGHT AS SIGNAL_WEIGHT,
    sd.DESCRIPTION AS SIGNAL_DESCRIPTION,
    ph.HEALTH_SCORE AS PRODUCT_HEALTH_SCORE,
    ph.HEALTH_STATUS AS PRODUCT_HEALTH_STATUS,
    ph.GREEN_COUNT AS PRODUCT_GREEN_COUNT,
    ph.YELLOW_COUNT AS PRODUCT_YELLOW_COUNT,
    ph.RED_COUNT AS PRODUCT_RED_COUNT
FROM DQ_SIGNAL_RESULTS sr
JOIN DQ_SIGNAL_DEFINITIONS sd ON sr.SIGNAL_ID = sd.SIGNAL_ID
JOIN DQ_DATA_PRODUCTS dp ON sr.PRODUCT_ID = dp.PRODUCT_ID
JOIN DQ_DOMAINS dom ON dp.DOMAIN_ID = dom.DOMAIN_ID
LEFT JOIN DQ_PRODUCT_HEALTH ph ON sr.PRODUCT_ID = ph.PRODUCT_ID AND sr.RUN_TS = ph.RUN_TS;
```

### Step 6 — Deploy the Dashboard

The Streamlit app queries `VW_DQ_METRICS` and works against any dataset that follows this schema.

```bash
cd streamlit_app
snow streamlit deploy --replace
```

### Key Technical Notes

- **Snowflake-hosted Streamlit** uses `get_active_session()` from `snowflake.snowpark.context`, not `st.connection()`
- Dependencies go in `environment.yml` (Anaconda channel), not `pyproject.toml`
- Navigation uses `st.sidebar.radio` + `importlib.import_module` (not `st.navigation` which requires newer Streamlit)
- All queries use `session.sql("...").to_pandas()`

---

## Repository Structure

```
dqdemo/
├── snowflake_native_dq_architecture.md    # Full architecture document
├── vertiv_dq_dashboard_guide.md           # This file
├── streamlit_app/
│   ├── streamlit_app.py                   # Main entry point
│   ├── snowflake.yml                      # Snowflake deployment config
│   ├── environment.yml                    # Anaconda dependencies
│   ├── app_pages/
│   │   ├── __init__.py
│   │   ├── overview.py                    # Overview page
│   │   ├── product_detail.py              # Product drill-down
│   │   ├── known_gaps.py                  # Gap tracking
│   │   └── ownership.py                   # Ownership & actions
```

---

## Links

- **Architecture Document:** `snowflake_native_dq_architecture.md` in this repo
- **GitHub:** [github.com/mstjohn79/dqarchitecture](https://github.com/mstjohn79/dqarchitecture)
- **Deployed App:** Available in Snowflake under `EPLAYGROUND.VERTIV_DQ.VERTIV_DQ_DASHBOARD`
