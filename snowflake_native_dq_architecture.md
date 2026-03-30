# Snowflake-Native Agentic Data Quality Architecture

## Replacing LangChain/CrewAI with Cortex Agents & Native Tooling

**Purpose:** This document maps the proposed "Autonomous Data Quality Agentic Mesh" (5-agent LangChain/CrewAI architecture) to a fully Snowflake-native implementation. The goal is to keep the entire solution — data plane *and* control plane — inside Snowflake.

---

## Executive Summary

The original draft proposes 5 AI agents orchestrated by LangChain/CrewAI/LangGraph running outside Snowflake, connecting back in via tool adapters. This creates operational complexity (external compute, connector maintenance, credential management, separate deployment pipeline) without clear benefit over native capabilities.

**Snowflake now provides all the building blocks natively:**

| Capability | External Approach | Snowflake-Native Replacement |
|---|---|---|
| Agent orchestration | LangChain / CrewAI / LangGraph | **Cortex Agents** |
| LLM reasoning | External LLM API calls | **Cortex AI Functions** (`AI_COMPLETE`, `AI_EXTRACT`, etc.) |
| Data profiling engine | Custom Python agents | **Data Metric Functions (DMFs) + INFORMATION_SCHEMA + Cortex AI** |
| Scheduling & execution | External cron / Airflow | **Snowflake Tasks** |
| NL query over DQ metrics | Custom RAG pipeline | **Cortex Analyst + Semantic Model** |
| Visualization | Custom dashboards | **Streamlit in Snowflake** |
| Human-in-the-loop | External approval UI | **Streamlit app + approval tables** |

---

## Data Profiling — Native in Snowflake

Data profiling is the foundation of the entire DQ pipeline. Before you can write rules, set thresholds, or score health, you need to understand what's actually in your tables. Snowflake provides profiling at three levels — all native, no external tools required.

### Level 1: Metadata Profiling (Zero-Cost)

Instant structural scan — column names, data types, nullable flags, row counts — pulled from metadata with no warehouse compute.

| Technique | What You Get |
|---|---|
| `DESCRIBE TABLE <table>` | Column names, types, nullable, default values, primary key, unique key |
| `INFORMATION_SCHEMA.COLUMNS` | Same metadata queryable across all tables in a schema — useful for bulk profiling |
| `INFORMATION_SCHEMA.TABLES` | Row counts, byte sizes, creation dates, last altered timestamps |
| `SHOW COLUMNS IN SCHEMA <schema>` | Quick column inventory across all tables |

**Example: Profile all columns in a schema**

```sql
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE, IS_NULLABLE,
       COLUMN_DEFAULT, COMMENT
FROM EPLAYGROUND.INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'VERTIV_DQ'
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

### Level 2: Statistical Profiling (System DMFs + SQL)

Compute-based profiling that calculates actual data distributions, cardinality, null rates, and freshness. This is where Snowflake's **system Data Metric Functions** shine — they're purpose-built for exactly this.

| System DMF | Profile Metric | DAMA Dimension |
|---|---|---|
| `SNOWFLAKE.CORE.NULL_COUNT` | Count of NULLs per column | Completeness |
| `SNOWFLAKE.CORE.UNIQUE_COUNT` | Count of distinct values per column | Uniqueness |
| `SNOWFLAKE.CORE.DUPLICATE_COUNT` | Count of duplicate values per column | Uniqueness |
| `SNOWFLAKE.CORE.FRESHNESS` | Time since last DML operation | Timeliness |
| `SNOWFLAKE.CORE.ROW_COUNT` | Total row count | Completeness |

**These can be attached directly to tables and scheduled to run automatically:**

```sql
-- Attach null count profiling to SRC_ACCOUNTS.EMAIL
ALTER TABLE EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS
  ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.NULL_COUNT
  ON (EMAIL);

-- Attach uniqueness profiling to SRC_ACCOUNTS.ACCOUNT_ID
ALTER TABLE EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS
  ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.UNIQUE_COUNT
  ON (ACCOUNT_ID);

-- Attach freshness check to the table
ALTER TABLE EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS
  ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.FRESHNESS
  ON (LAST_MODIFIED_DATE);

-- Set schedule: run every hour
ALTER TABLE EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS
  SET DATA_METRIC_SCHEDULE = '60 MINUTE';
```

**Beyond DMFs — ad-hoc SQL profiling for deeper stats:**

```sql
-- Distribution profile for any column
SELECT
    COUNT(*)                                    AS total_rows,
    COUNT(DISTINCT ACCOUNT_NAME)                AS distinct_values,
    COUNT(*) - COUNT(ACCOUNT_NAME)              AS null_count,
    ROUND(100.0 * (COUNT(*) - COUNT(ACCOUNT_NAME)) / COUNT(*), 2) AS null_pct,
    MIN(LEN(ACCOUNT_NAME))                      AS min_length,
    MAX(LEN(ACCOUNT_NAME))                      AS max_length,
    ROUND(AVG(LEN(ACCOUNT_NAME)), 1)            AS avg_length
FROM EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS;
```

### Level 3: AI-Powered Profiling (Cortex AI Functions)

For profiling that goes beyond statistics — pattern inference, semantic classification, anomaly explanation — use Cortex AI functions. This is what replaces the "intelligent" profiling the PDF's Agent 1 was supposed to do.

| Cortex Function | Profiling Use Case |
|---|---|
| `AI_COMPLETE` | Feed sample data + schema to an LLM → get plain-language data characterization, pattern descriptions, quality risk assessment, KDE recommendations |
| `AI_CLASSIFY` | Classify columns by semantic type (PII, identifier, measure, date, categorical) — useful for auto-tagging and governance |
| `AI_EXTRACT` | Extract structured attributes from free-text columns (e.g., parse addresses, product codes, status values) |

**Example: LLM-powered table profile summary**

```sql
SELECT SNOWFLAKE.CORTEX.AI_COMPLETE(
  'snowflake-llama-3.3-70b',
  CONCAT(
    'You are a data quality analyst. Profile this table and identify:\n',
    '1. Which columns are likely Key Data Elements (KDEs)\n',
    '2. Columns with potential quality risks (high nulls, low cardinality, suspicious patterns)\n',
    '3. Recommended DAMA dimensions to monitor per column\n',
    '4. Suggested sampling strategy\n\n',
    'Table: SRC_ACCOUNTS (300 rows)\n',
    'Columns: ', (SELECT LISTAGG(COLUMN_NAME || ' (' || DATA_TYPE || ')', ', ')
                  FROM EPLAYGROUND.INFORMATION_SCHEMA.COLUMNS
                  WHERE TABLE_SCHEMA = 'VERTIV_DQ' AND TABLE_NAME = 'SRC_ACCOUNTS'),
    '\n\nSample rows (first 5):\n',
    (SELECT LISTAGG(TO_VARCHAR(OBJECT_CONSTRUCT(*)), '\n') 
     FROM (SELECT * FROM EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS LIMIT 5))
  )
) AS PROFILE_SUMMARY;
```

### Level 4: Custom Profiling DMFs (Domain-Specific)

Write custom DMFs for profiling checks that system DMFs don't cover — format validation, referential integrity, distribution anomalies.

```sql
-- Email format validity profiling
CREATE OR REPLACE DATA METRIC FUNCTION
  VERTIV_DQ.DMF_EMAIL_VALIDITY(ARG_T TABLE(ARG_C VARCHAR))
  RETURNS NUMBER AS
$$
  SELECT COUNT_IF(NOT RLIKE(ARG_C, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$'))
  FROM ARG_T
$$;

-- Referential integrity check (e.g., do all contacts reference a valid account?)
CREATE OR REPLACE DATA METRIC FUNCTION
  VERTIV_DQ.DMF_ORPHAN_COUNT(ARG_T TABLE(ARG_C VARCHAR))
  RETURNS NUMBER AS
$$
  SELECT COUNT(*)
  FROM ARG_T
  WHERE ARG_C NOT IN (SELECT ACCOUNT_ID FROM EPLAYGROUND.VERTIV_DQ.SRC_ACCOUNTS)
    AND ARG_C IS NOT NULL
$$;

-- Z-score anomaly detection on numeric columns
CREATE OR REPLACE DATA METRIC FUNCTION
  VERTIV_DQ.DMF_ZSCORE_ANOMALY(ARG_T TABLE(ARG_C NUMBER))
  RETURNS NUMBER AS
$$
  SELECT COUNT_IF(ABS((ARG_C - AVG(ARG_C) OVER()) / NULLIF(STDDEV(ARG_C) OVER(), 0)) > 3)
  FROM ARG_T
$$;
```

### How Profiling Fits in the Pipeline

```
┌──────────────┐    ┌──────────────────┐    ┌───────────────┐    ┌──────────────┐
│  Level 1     │    │  Level 2         │    │  Level 3      │    │  Level 4     │
│  Metadata    │───▶│  System DMFs     │───▶│  AI Profiling │───▶│  Custom DMFs │
│  Scan        │    │  (scheduled)     │    │  (on-demand)  │    │  (scheduled) │
│              │    │                  │    │               │    │              │
│  DESCRIBE    │    │  NULL_COUNT      │    │  AI_COMPLETE  │    │  Format      │
│  INFO_SCHEMA │    │  UNIQUE_COUNT    │    │  AI_CLASSIFY  │    │  Referential │
│              │    │  DUPLICATE_COUNT │    │  AI_EXTRACT   │    │  Statistical │
│              │    │  FRESHNESS       │    │               │    │  Anomaly     │
└──────────────┘    └──────────────────┘    └───────────────┘    └──────────────┘
       │                    │                       │                    │
       └────────────────────┴───────────────────────┴────────────────────┘
                                       │
                              ┌────────▼────────┐
                              │  DQ_SIGNAL_     │
                              │  RESULTS        │
                              │  (all profiling │
                              │  results land   │
                              │  here)          │
                              └─────────────────┘
```

**Profiling is not a separate step — it IS the DQ Engine.** Levels 1-2 run automatically on schedule via DMFs and Tasks. Level 3 runs on-demand when Agent 1 (Base Prep) characterizes a new table. Level 4 runs on schedule alongside Level 2 for ongoing domain-specific monitoring.

---

## Agent-by-Agent Mapping

### Agent 1: DQ Base Prep
*Original: Scan tables, sample data, recommend Key Data Elements (KDEs), determine sampling strategy*

#### Snowflake-Native Implementation

| Component | How It Works |
|---|---|
| **INFORMATION_SCHEMA / DESCRIBE TABLE** | Enumerate columns, data types, and table metadata for any target data asset |
| **System DMFs** (`NULL_COUNT`, `UNIQUE_COUNT`, `DUPLICATE_COUNT`) | Automated statistical profiling — no custom code needed |
| **Cortex `AI_COMPLETE`** | Feed table schemas + sample rows to an LLM (e.g., `snowflake-llama-3.3-70b`) and prompt it to recommend which columns qualify as KDEs and what sampling % is representative |
| **Cortex `AI_CLASSIFY`** | Classify columns by semantic type (PII, identifier, measure, etc.) to assist KDE selection |
| **Output** | Store recommendations in `DQ_SIGNAL_DEFINITIONS` (existing table) or a new `DQ_KDE_RECOMMENDATIONS` staging table |

**Why this works:** The "agent" behavior here is mostly deterministic metadata scanning plus one LLM reasoning call. A stored procedure that queries `INFORMATION_SCHEMA`, runs sample profiling SQL, and calls `AI_COMPLETE` for KDE recommendations handles this end-to-end without an external agent framework.

---

### Agent 2: DQ Engine
*Original: SQL profiling on KDEs, generate DQ metric scores, anomaly detection (statistical distribution, pattern inference)*

#### Snowflake-Native Implementation

| Component | How It Works |
|---|---|
| **Data Metric Functions (DMFs)** | System and custom DMFs attached directly to tables. System DMFs cover null counts, uniqueness, freshness out of the box. Custom DMFs (SQL or Python) handle domain-specific checks |
| **Custom DMFs in Python** | For statistical anomaly detection — z-score, IQR, distribution checks — write Python UDTFs that run inside Snowflake compute |
| **Snowpark Stored Procedures** | Orchestrate multi-step profiling logic (e.g., run all signals for a product, compute weighted health scores) |
| **Snowflake Tasks** | Schedule execution — daily, hourly, or event-driven via streams |
| **Output** | Results persist to `DQ_SIGNAL_RESULTS` and `DQ_PRODUCT_HEALTH` (existing tables) |

**Why this works:** This agent is entirely deterministic SQL/Python execution. DMFs are purpose-built for exactly this — they run profiling checks, persist results, and integrate with Snowflake's monitoring infrastructure. There is no LLM reasoning needed here, so wrapping it in a LangChain agent adds overhead with no benefit.

#### Example: Custom DMF for Anomaly Detection

```sql
CREATE OR REPLACE DATA METRIC FUNCTION
  VERTIV_DQ.DQ_ZSCORE_ANOMALY(ARG_T TABLE(ARG_C NUMBER))
  RETURNS NUMBER AS
$$
  SELECT CASE
    WHEN ABS(AVG(ARG_C) - (SELECT AVG(ARG_C) FROM ARG_T)) 
          / NULLIF(STDDEV(ARG_C), 0) > 3
    THEN 1 ELSE 0
  END
  FROM ARG_T
$$;
```

---

### Agent 3: DQ Reviewer (Reasoning Agent)
*Original: LLM chain-of-thought review of DQ scores and anomalies, generate suggestions, validate accuracy, severity-based recommendations*

#### Snowflake-Native Implementation

**This is where Cortex Agents provide the most value.** This is the one agent that genuinely requires LLM reasoning.

| Component | How It Works |
|---|---|
| **Cortex Agent** | A deployed agent with instructions encoding the DQ review logic |
| **Cortex Analyst Tool** | Attached to the `dq_semantic_model.yaml` — gives the agent structured access to all DQ metrics, thresholds, trends, domains, and products |
| **SQL Tool** | Allows the agent to write and execute ad-hoc investigative SQL when the semantic model doesn't cover a specific drill-down |
| **Agent Instructions** | Encode the chain-of-thought: examine RED/YELLOW signals → compare to 7-day and 30-day trends → cross-reference related signals in same domain → assess severity → generate plain-language root cause explanation → recommend actions with owners |
| **Output** | Agent responses stored in `DQ_AGENT_REVIEWS` table for audit trail |

#### Cortex Agent Configuration (Conceptual)

```yaml
name: dq_reviewer_agent
model: snowflake-llama-3.3-70b
instructions: |
  You are a Data Quality Reviewer for Vertiv. Your job is to analyze 
  DQ signal results and provide chain-of-thought reasoning about:
  
  1. Which signals are RED or YELLOW and why
  2. Whether the degradation is new (compare to last 7 and 30 days)
  3. Cross-domain impact (e.g., does a RED in Sustainability affect CIA?)
  4. Root cause hypotheses based on source system and signal type
  5. Severity ranking: CRITICAL / HIGH / MEDIUM
  6. Recommended actions and responsible owners
  
  Always reference specific signal names, values, and thresholds.
  Always check THRESHOLD_DIRECTION before interpreting values.

tools:
  - type: cortex_analyst
    semantic_model: dq_semantic_model.yaml
  - type: sql
    description: Run investigative SQL against EPLAYGROUND.VERTIV_DQ
```

**Why this works:** Cortex Agents natively support tool use (Cortex Analyst for structured queries, SQL for ad-hoc), maintain conversation context, and run entirely inside Snowflake. This replaces the LangChain agent + Snowflake connector + external LLM API pattern with a single managed service.

---

### Agent 4: DQ Metrics Finalizer
*Original: Re-run DQ metrics for high-severity recommendations*

#### Snowflake-Native Implementation

| Component | How It Works |
|---|---|
| **Stored Procedure** | Reads `DQ_AGENT_REVIEWS`, filters for CRITICAL/HIGH severity recommendations that suggest re-measurement, re-executes the relevant `CALCULATION_SQL` from `DQ_SIGNAL_DEFINITIONS` |
| **Snowflake Task** | Triggers the finalizer procedure after the reviewer agent completes |
| **Conditional logic** | Simple SQL `CASE` / `IF` branching — if the reviewer flagged a signal for re-run, execute it; otherwise skip |

**Why this works:** This is orchestration logic, not AI reasoning. A stored procedure with conditional branching is simpler, faster, more auditable, and more reliable than an LLM agent deciding whether to re-run a query. The LLM already made the decision in Agent 3 — Agent 4 just executes it.

#### Example: Finalizer Procedure (Simplified)

```sql
CREATE OR REPLACE PROCEDURE VERTIV_DQ.SP_FINALIZE_DQ_METRICS()
RETURNS VARCHAR
LANGUAGE SQL
AS
BEGIN
  -- Get high-severity re-run recommendations from latest review
  LET c1 CURSOR FOR
    SELECT sd.SIGNAL_ID, sd.CALCULATION_SQL
    FROM DQ_AGENT_REVIEWS ar
    JOIN DQ_SIGNAL_DEFINITIONS sd ON ar.SIGNAL_ID = sd.SIGNAL_ID
    WHERE ar.SEVERITY IN ('CRITICAL', 'HIGH')
      AND ar.ACTION_TYPE = 'RE_MEASURE'
      AND ar.REVIEW_DATE = CURRENT_DATE();
  
  FOR rec IN c1 DO
    EXECUTE IMMEDIATE :rec.CALCULATION_SQL;
  END FOR;
  
  RETURN 'Finalization complete';
END;
```

---

### Agent 5: DQ Visualizer
*Original: Roll up scores, generate drill-down BI report*

#### Snowflake-Native Implementation

| Component | How It Works |
|---|---|
| **Streamlit in Snowflake** (Phase 1) | Domain selector → product list → signal detail → trend charts. Already scaffolded in `streamlit_app/` |
| **Cortex Analyst / Snowflake Intelligence** | Natural-language drill-down: "Why is BOM Health red?", "What changed this week?", "Which signals are driving risk in CIA?" |
| **Semantic View** | Built from `dq_semantic_model.yaml` — gives Cortex Analyst structured knowledge of the entire DQ data model |
| **React app** (Phase 2) | Enterprise UI with embedded Snowflake Intelligence for the same NLQ capability |

**Why this works:** Visualization is a UI concern, not an agent concern. The PDF itself flags this as "optional — might be better to build using other capabilities." Streamlit + Cortex Analyst is the right answer.

---

### Human-in-the-Loop

| Component | How It Works |
|---|---|
| **Streamlit approval UI** | Data steward sees agent recommendations, reviews reasoning, clicks approve/reject, writes rationale |
| **`DQ_APPROVALS` table** | Persists approval decisions, timestamps, reviewer identity, rejection reasons |
| **Cortex Agent conversations** | Stewards can interact with the DQ reviewer agent conversationally — challenge findings, ask for more detail, request alternate analysis |

---

## Revised Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        SNOWFLAKE (Everything)                        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  DATA LAYER                                                    │  │
│  │  • Source tables (Sustainability, CIA, Marketing, BOM)         │  │
│  │  • DQ_DOMAINS, DQ_DATA_PRODUCTS                                │  │
│  │  • DQ_SIGNAL_DEFINITIONS (rules, thresholds, SQL)              │  │
│  │  • DQ_SIGNAL_RESULTS (time-series measurements)                │  │
│  │  • DQ_PRODUCT_HEALTH (weighted roll-ups)                       │  │
│  │  • DQ_AGENT_REVIEWS (LLM reasoning audit trail)                │  │
│  │  • DQ_APPROVALS (human-in-the-loop decisions)                  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  PROFILING LAYER (foundation — feeds everything below)         │  │
│  │                                                                │  │
│  │  Level 1: DESCRIBE / INFORMATION_SCHEMA (metadata, zero-cost) │  │
│  │  Level 2: System DMFs — NULL_COUNT, UNIQUE_COUNT, FRESHNESS   │  │
│  │           (attached to tables, scheduled, automatic)           │  │
│  │  Level 3: Cortex AI — AI_COMPLETE, AI_CLASSIFY, AI_EXTRACT    │  │
│  │           (LLM-powered profiling: KDEs, patterns, risk)       │  │
│  │  Level 4: Custom DMFs — format, referential, z-score, IQR     │  │
│  │           (domain-specific, scheduled alongside Level 2)      │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              ▼                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  COMPUTE LAYER                                                 │  │
│  │                                                                │  │
│  │  Deterministic (replaces Agents 1, 2, 4):                      │  │
│  │  • Data Metric Functions (system + custom)                     │  │
│  │  • Stored Procedures (scoring, finalization, re-runs)          │  │
│  │  • Snowflake Tasks (scheduling, orchestration, DAGs)           │  │
│  │  • Cortex AI Functions (AI_COMPLETE for KDE recommendation)    │  │
│  │                                                                │  │
│  │  Reasoning (replaces Agent 3):                                 │  │
│  │  • Cortex Agent (DQ Reviewer)                                  │  │
│  │    ├── Tool: Cortex Analyst (semantic model)                   │  │
│  │    └── Tool: SQL (ad-hoc investigation)                        │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  PRESENTATION LAYER (replaces Agent 5):                        │  │
│  │  • Streamlit in Snowflake (Phase 1 - Crawl)                   │  │
│  │  • React + Snowflake Intelligence (Phase 2 - Walk)            │  │
│  │  • Cortex Agent chat interface (Phase 3 - Run)                │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## What This Eliminates

| External Component (PDF Proposal) | Status |
|---|---|
| LangChain / CrewAI / LangGraph orchestrator | **Eliminated** — Cortex Agents + Tasks |
| External compute environment | **Eliminated** — runs on Snowflake warehouses |
| Snowflake connector / tool adapters | **Eliminated** — agents have native SQL access |
| External LLM API calls | **Eliminated** — Cortex LLMs run inside Snowflake |
| Separate state / memory store | **Eliminated** — Snowflake tables (already built) |
| Credential management for external services | **Eliminated** — RBAC handles everything |

---

## What We Keep From the Original Proposal

- **Data model** — `DQ_DOMAINS`, `DQ_DATA_PRODUCTS`, `DQ_SIGNAL_DEFINITIONS`, `DQ_SIGNAL_RESULTS`, `DQ_PRODUCT_HEALTH` are already built and sound
- **6 DAMA dimensions** — Completeness, Consistency, Timeliness, Validity, Uniqueness, Accuracy — already encoded in the semantic model
- **Phased approach** — Crawl (Streamlit) → Walk (React + Intelligence) → Run (Cortex Agents) remains the right progression
- **Weighted health scoring** — GREEN/YELLOW/RED thresholds with weighted roll-ups are already implemented
- **Human-in-the-loop** — steward approval before publishing remains important

---

## Key Architectural Insight

The PDF assumes **5 AI agents** are needed. In reality:

- **Agents 1, 2, and 4** are deterministic work (scan metadata, run SQL, re-run SQL). These belong in **DMFs, stored procedures, and Tasks** — not LLM agents. Wrapping deterministic logic in an LLM agent makes it slower, less predictable, and harder to debug.
- **Agent 3** is the only one that genuinely requires LLM reasoning (interpret scores, explain root causes, recommend actions). This maps cleanly to **one Cortex Agent**.
- **Agent 5** is a UI concern that maps to **Streamlit + Cortex Analyst**.

**Net result: 5 external agents → 1 Cortex Agent + native infrastructure. Zero external dependencies.**

---

## Implementation Steps

### Step 0: Data Profiling Baseline
- Run Level 1 metadata scan across all `SRC_*` tables via `INFORMATION_SCHEMA.COLUMNS`
- Attach system DMFs (`NULL_COUNT`, `UNIQUE_COUNT`, `DUPLICATE_COUNT`, `FRESHNESS`) to all source tables for Level 2 statistical profiling
- Set `DATA_METRIC_SCHEDULE` on each source table (e.g., `60 MINUTE`)
- Run Level 3 AI profiling (`AI_COMPLETE`) on each table to generate KDE recommendations and risk assessments
- Review profiling output to inform signal definitions and thresholds

### Step 1: Custom Data Metric Functions
- Write custom DMFs for domain-specific checks (email format, referential integrity, z-score anomaly detection)
- Attach custom DMFs to source tables alongside system DMFs
- Validate all DMF results land in `DQ_SIGNAL_RESULTS`
- Schedule via Snowflake Tasks

### Step 2: Semantic View
- Deploy `dq_semantic_model.yaml` as a Snowflake Semantic View
- Validate with Cortex Analyst queries ("show me all RED signals", "BOM Health trend last 30 days")

### Step 3: Cortex Agent (DQ Reviewer)
- Create agent with Cortex Analyst tool (pointing to semantic view) and SQL tool
- Encode review instructions (chain-of-thought, severity ranking, action recommendations)
- Test with current DQ data

### Step 4: Streamlit App (Phase 1 UI)
- Domain overview → product detail → signal drill-down → trend charts
- Embed Cortex Agent chat for natural-language DQ exploration
- Add approval workflow for human-in-the-loop

### Step 5: Orchestration
- Task DAG: Run DMFs → compute health scores → trigger Cortex Agent review → store results → notify stewards
- Finalizer procedure for high-severity re-runs

---

*This architecture keeps all data, compute, reasoning, and presentation inside Snowflake — no external agent frameworks, no connector maintenance, no credential sprawl, and full RBAC governance.*
