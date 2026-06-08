# Data Quality on Snowflake: Crawl, Walk, Run Framework

**Prepared by:** Evolv Solutions Engineering  
**Platform:** Snowflake + Cortex Code  
**Last Updated:** June 2026

---

## Executive Summary

This document outlines a progressive data quality framework built on Snowflake's native **Data Metric Functions (DMFs)**, organized into three maturity tiers. Each tier builds on the previous, allowing teams to start with zero-configuration quality checks and mature into fully automated data governance with pipeline protection.

| Tier | What You Get | Setup Required |
|------|-------------|----------------|
| **Crawl** | Ad-hoc quality checks, table comparisons, usage analytics | None |
| **Walk** | Continuous monitoring, health scores, trend analysis, custom rules | DMFs deployed |
| **Run** | Automated alerts, circuit breakers, incident investigation, cost optimization | DMFs + alerts + cross-system orchestration |

---

## Tier 1: CRAWL — Foundation & Awareness

### Goal
Answer the question: *"Can I trust my data right now?"* — with zero setup.

### Capabilities

#### 1. Ad-Hoc Quality Assessment
- One-time quality snapshot using inline `SNOWFLAKE.CORE.*` functions
- No DMF setup required — works on any table, schema, or Marketplace listing
- Checks: null counts, uniqueness, freshness, completeness, value distributions
- Use case: Quick pre-meeting data validation, new dataset evaluation, listing quality check

#### 2. Dataset Popularity & Usage Analytics
- Identifies most-used and least-used tables via `ACCOUNT_USAGE`
- Surfaces stale/unused objects with storage cost estimates
- Answers: "What should we monitor first?" and "What can we deprecate?"
- Helps prioritize which tables deserve investment in Tier 2 monitoring

#### 3. Table Comparison & Migration Validation
- Point-in-time data diff between any two tables (dev vs prod, pre vs post migration)
- Sub-workflows available:
  - **Summary Diff** — Quick row count deltas
  - **Row-Level Diff** — Actual added/removed/modified rows
  - **Schema Comparison** — DDL structure differences
  - **Distribution Analysis** — Statistical distribution shifts (numeric + categorical)
  - **Validation Report** — Full migration sign-off document

### Collateral Built

| Asset | Description |
|-------|-------------|
| Ad-hoc assessment workflow | Orchestration logic for one-time scans |
| `adhoc-column-quality.sql` | Parameterized SQL using inline system functions |
| Popularity workflow | Usage-based prioritization logic |
| Compare-tables workflow suite (5 sub-workflows) | Full comparison toolkit |
| 10 compare-tables SQL templates | Schema diff, row counts, added/removed/modified rows, distributions |
| Compare-tables principles doc | 12 operating principles for reliable comparisons |
| Compare-tables concepts reference | Tool usage patterns and methodology |

### When to Use
- First engagement with a new client/dataset
- Pre-migration validation
- Evaluating Marketplace listings before purchase
- Building the business case for continuous monitoring (Tier 2)

---

## Tier 2: WALK — Continuous Monitoring

### Goal
Answer: *"Is my data quality getting better or worse over time?"* — with scheduled, automated measurement.

### Capabilities

#### 1. Monitor Recommendations & Deployment
- Profiles all columns in a schema (data types, null rates, cardinality, patterns)
- Ranks tables/columns by monitoring criticality
- Generates ready-to-execute DDL for attaching the right DMFs
- Supports schema-level bulk deployment (ROW_COUNT + FRESHNESS on all tables at once)

#### 2. Health Scoring
- Single schema-wide health percentage (passing metrics / total metrics)
- Per-table and per-column drill-down
- Real-time and historical views

#### 3. Root Cause Analysis
- Surfaces which specific metrics are failing and on which columns
- Provides actionable fix recommendations
- Identifies whether failures are new or persistent

#### 4. Regression Detection
- Compares current measurement run against previous run
- Highlights new failures, resolved issues, and health delta
- Catches quality degradation as soon as it happens

#### 5. Trend Analysis
- 30-day time-series of quality scores
- Identifies persistent vs transient issues
- Shows whether quality is trending up or down

#### 6. Expectations Management
- Set pass/fail thresholds on any DMF (e.g., "NULL rate must be < 5%")
- Review current expectation inventory and violation status
- Tune thresholds based on observed patterns

#### 7. Custom Data Metric Functions
- Format validation (email, phone, date patterns)
- Value range checks (min/max boundaries)
- Cross-column validation (e.g., start_date < end_date)
- Referential integrity (orphaned FK rows)
- Accepted values / categorical validation

#### 8. Within-Group (Segmented) Monitoring
- Per-group metrics via `WITHIN GROUP` clause
- Monitor quality separately by region, category, business unit, etc.
- No need for separate DMF associations per segment

### Collateral Built

| Asset | Description |
|-------|-------------|
| Monitor recommendations workflow | Column profiling + ranked DMF deployment |
| `monitor-recommendations.sql` | Profiling and coverage analysis template |
| Health scoring workflow | Schema health % calculation and presentation |
| `schema-health-snapshot-realtime.sql` | Primary health score template |
| `schema-health-snapshot.sql` | Fallback health score template |
| Root cause analysis workflow | Failure identification + fix suggestions |
| `schema-root-cause-realtime.sql` / `schema-root-cause.sql` | Root cause templates (primary + fallback) |
| Regression detection workflow | Run-over-run comparison logic |
| `schema-regression-detection.sql` | Regression comparison template |
| Trend analysis workflow | 30-day time-series logic |
| `schema-quality-trends.sql` | Trend analysis template |
| Expectations management workflow | Threshold setting and tuning |
| `expectations-review.sql` | Expectation inventory + pass/fail status |
| Custom DMF patterns workflow | Format/range/FK/cross-column validation |
| `custom-dmf-create.sql` | CREATE DATA METRIC FUNCTION templates |
| Within-group DMF workflow | Per-group segmented monitoring |
| `within-group-dmf.sql` | WITHIN GROUP ALTER TABLE DDL |
| `schema-level-dmf.sql` | Bulk-attach DMFs to entire schema |
| DMF concepts reference (769 lines) | Complete technical reference for DMF setup, scheduling, syntax |

### When to Use
- After Tier 1 identifies which tables matter most
- When the team wants ongoing visibility into data health
- When data quality SLAs exist but aren't yet automated
- When custom business rules need enforcement (format, range, FK integrity)

---

## Tier 3: RUN — Automated Governance & Remediation

### Goal
Answer: *"Bad data should never reach production — automate protection and alert me."*

### Capabilities

#### 1. SLA Alerting
- Creates Snowflake ALERT objects that fire when quality drops below thresholds
- Configurable notification channels (email, webhook, Slack via integration)
- Logs all violations to a dedicated tracking table
- Monitoring instructions included for operational handoff

#### 2. Circuit Breaker
- Automatically halts downstream pipelines (SUSPEND TASK) when upstream quality fails
- Uses `DATA_QUALITY_MONITORING_EXPECTATION_STATUS` + `expectation_violated` as trigger
- Includes resume workflow for re-enabling pipelines after fixes
- Prevents bad data from propagating through the entire pipeline

#### 3. Coverage Gaps & Meta-Observability
- Monitors the monitors themselves:
  - **Coverage %** — What fraction of critical tables have DMFs?
  - **Noisy monitors** — Firing too often (threshold too tight)
  - **Silent monitors** — Never firing (threshold too loose or redundant)
  - **Cost analysis** — DMF credit consumption and optimization opportunities
- Ensures monitoring investment is effective and efficient

#### 4. DQ Incident Investigation (Multi-Skill Orchestration)
- Full root-cause analysis that spans multiple Snowflake capabilities:
  - **Data Quality skill** → Identify the violation and timeline
  - **Lineage skill** → Trace upstream to find where bad data originated
  - **Data Governance skill** → Check query history and task failures
- Produces multi-dimensional incident report with:
  - Timeline of the quality drop
  - Primary cause identification
  - Contributing factors
  - Remediation steps

### Collateral Built

| Asset | Description |
|-------|-------------|
| SLA alerting workflow | Alert creation with approval gates |
| `schema-sla-alert.sql` | CREATE ALERT DDL template |
| Circuit breaker workflow | Pipeline protection with auto-suspend |
| `circuit-breaker-setup.sql` | ALERT + TASK suspension template |
| Coverage gaps workflow | Meta-observability analysis |
| `coverage-gaps-summary.sql` | Coverage % + unmonitored tables template |
| `monitor-effectiveness.sql` | Noisy/silent monitor detection template |
| DQ incident investigation workflow | Multi-skill orchestrated root cause |
| Cross-skill delegation rules | Explicit handoff patterns to lineage and governance skills |

### When to Use
- Production pipelines where bad data has downstream cost
- Regulated environments requiring automated enforcement
- Teams with data quality SLAs tied to business outcomes
- Organizations needing to optimize their monitoring spend

---

## Bonus: Prompt Quality (Standalone)

Separate from the DMF maturity path, we also built prompt quality tooling:

| Workflow | Description |
|---------|-------------|
| Prompt Quality Scoring | Scores any LLM prompt on 9 dimensions (1-10 scale) via Cortex Complete |
| Prompt Improvement | Rewrites weak prompts + re-scores to show improvement delta |
| Prompt Execute & Compare | A/B tests original vs improved prompt output side-by-side |

Supporting reference: 9-dimension scoring rubric with sub-criteria and scoring rules.

---

## Architecture & Design Patterns

### Decision Tree Router
A comprehensive decision tree in the main skill file routes user intent to the correct workflow *before* any SQL runs. Users don't need to know which tier they're in — the system detects it automatically.

### Preflight Check Gate
Every DMF-based workflow validates the environment first:
- If `total_dmfs_attached = 0` → routes to Crawl options
- If DMFs present but no results yet → advises to wait
- If results available → proceeds to Walk/Run workflows

### Mandatory Approval Gates
All write operations pause for explicit user approval:
- Before deploying DMF recommendations
- Before creating alerts
- Before activating circuit breakers
- Before creating custom DMFs

### Template + Workflow Separation
- **SQL templates** are parameterized with `<database>` and `<schema>` placeholders
- **Workflows** contain orchestration logic, presentation format, and decision points
- This separation allows templates to be reused across workflows and customized per engagement

---

## Total Collateral Inventory

| Category | Count | Description |
|----------|-------|-------------|
| Workflows | 19 | Orchestration logic for each capability |
| SQL Templates | 31 | Parameterized, ready-to-execute Snowflake SQL |
| Reference Docs | 5 | Technical concepts, principles, scoring rubrics |
| Router/Entry Point | 1 | Decision tree + preflight check logic |
| **Total** | **57 files** | Complete framework |

---

## How to Use This With a Client

1. **Start at Crawl** — Run an ad-hoc assessment on their most critical schema. Show them what quality looks like today with zero setup. Use popularity analysis to identify what matters most.

2. **Propose Walk** — Based on Crawl findings, recommend DMF deployment on high-priority tables. Show the health scoring and trend analysis they'll get in return.

3. **Graduate to Run** — Once monitoring is stable and patterns are understood, layer in SLA alerts, circuit breakers, and coverage gap analysis for full automated governance.

Each tier builds on the previous — nothing is throwaway work.

---

## Snowflake Features Leveraged

- Data Metric Functions (system + custom)
- `SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_RESULTS()` (table function)
- `SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_EXPECTATION_STATUS` (view)
- `INFORMATION_SCHEMA.DATA_METRIC_FUNCTION_REFERENCES()` (table function)
- `SNOWFLAKE.CORE.*` inline functions (NULL_COUNT, DUPLICATE_COUNT, FRESHNESS, etc.)
- Schema-level DMF associations (`ALTER SCHEMA ADD DATA METRIC FUNCTION`)
- `WITHIN GROUP` clause for segmented monitoring
- Snowflake ALERTs for SLA monitoring
- Snowflake TASKs (suspend/resume for circuit breakers)
- `ACCOUNT_USAGE` views for popularity and cost analysis
- Cortex Complete for prompt quality scoring
