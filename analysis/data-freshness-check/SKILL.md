---
name: data-freshness-check
description: Check when tables were last updated and flag stale tables that may have missed their freshness SLA.
---

# Data Freshness Check

## Goal
Quickly answer:
- **When was this table last updated** (last DML activity)?
- Which tables have **not received any data** in the expected timeframe?
- Are my mart/reporting tables **fresh enough** for dashboards?

## Inputs to collect
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `table_name`
- Optional: `stale_threshold_hours` (default 24) -- tables with no DML in this window are flagged stale
- Optional: `limit` (optional, default 50)

## Workflow

### 1) Last DML activity per table

> **Note:** `TABLE_DML_HISTORY` has up to **6 hours** latency and reports in **hourly buckets**.
> It excludes hybrid tables and background maintenance (auto-clustering, MV refresh).
> For very recent activity (within the latency window), this may not yet show the latest writes.

```sql
SELECT
  database_name,
  schema_name,
  table_name,
  MAX(end_time) AS last_dml_at,
  DATEDIFF('hour', MAX(end_time), CURRENT_TIMESTAMP()) AS hours_since_last_dml,
  SUM(rows_added) AS total_rows_added,
  SUM(rows_removed) AS total_rows_removed,
  SUM(rows_updated) AS total_rows_updated
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY database_name, schema_name, table_name
ORDER BY last_dml_at DESC
LIMIT <LIMIT>;
```

### 2) Flag stale tables (no DML within threshold)
Tables that exist but have not received any DML activity within the stale threshold.

```sql
WITH last_activity AS (
  SELECT
    database_name,
    schema_name,
    table_name,
    table_id,
    MAX(end_time) AS last_dml_at
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY
  WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  GROUP BY database_name, schema_name, table_name, table_id
)
SELECT
  t.table_catalog  AS database_name,
  t.table_schema   AS schema_name,
  t.table_name,
  t.table_type,
  a.last_dml_at,
  DATEDIFF('hour', a.last_dml_at, CURRENT_TIMESTAMP()) AS hours_since_last_dml,
  CASE
    WHEN a.last_dml_at IS NULL THEN 'NO_DML_IN_30_DAYS'
    WHEN DATEDIFF('hour', a.last_dml_at, CURRENT_TIMESTAMP()) > <STALE_THRESHOLD_HOURS> THEN 'STALE'
    ELSE 'FRESH'
  END AS freshness_status
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
LEFT JOIN last_activity a
  ON a.database_name = t.table_catalog
 AND a.schema_name   = t.table_schema
 AND a.table_name    = t.table_name
WHERE t.deleted IS NULL
  AND t.table_type IN ('BASE TABLE', 'DYNAMIC TABLE')
  AND (<DATABASE_NAME_IS_NULL> OR t.table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR t.table_schema = '<SCHEMA_NAME>')
  AND (
    a.last_dml_at IS NULL
    OR DATEDIFF('hour', a.last_dml_at, CURRENT_TIMESTAMP()) > <STALE_THRESHOLD_HOURS>
  )
ORDER BY hours_since_last_dml DESC NULLS FIRST
LIMIT <LIMIT>;
```

### 3) Freshness summary for a specific schema
Quick dashboard-friendly view of all tables in a schema with their freshness status.

```sql
WITH last_activity AS (
  SELECT
    database_name,
    schema_name,
    table_name,
    MAX(end_time) AS last_dml_at,
    SUM(rows_added) AS recent_rows_added
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY
  WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
    AND database_name = '<DATABASE_NAME>'
    AND schema_name = '<SCHEMA_NAME>'
  GROUP BY database_name, schema_name, table_name
)
SELECT
  t.table_name,
  t.table_type,
  a.last_dml_at,
  DATEDIFF('hour', a.last_dml_at, CURRENT_TIMESTAMP()) AS hours_since_last_dml,
  a.recent_rows_added,
  CASE
    WHEN a.last_dml_at IS NULL THEN 'NO_RECENT_DML'
    WHEN DATEDIFF('hour', a.last_dml_at, CURRENT_TIMESTAMP()) <= <STALE_THRESHOLD_HOURS> THEN 'FRESH'
    ELSE 'STALE'
  END AS freshness_status
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
LEFT JOIN last_activity a
  ON a.database_name = t.table_catalog
 AND a.schema_name   = t.table_schema
 AND a.table_name    = t.table_name
WHERE t.deleted IS NULL
  AND t.table_catalog = '<DATABASE_NAME>'
  AND t.table_schema  = '<SCHEMA_NAME>'
  AND t.table_type IN ('BASE TABLE', 'DYNAMIC TABLE')
ORDER BY
  CASE freshness_status
    WHEN 'NO_RECENT_DML' THEN 1
    WHEN 'STALE' THEN 2
    WHEN 'FRESH' THEN 3
  END,
  hours_since_last_dml DESC NULLS FIRST;
```

## Output format
Return:
1) Last DML timestamp per table (or for the specific table requested)
2) Freshness status: FRESH / STALE / NO_RECENT_DML
3) Hours since last DML activity
4) Recommendations:
   - If STALE: check upstream pipeline health with `monitor-task-failures` or `monitor-dynamic-table-refreshes`
   - If NO_RECENT_DML: table may be abandoned -- cross-reference with `find-unused-tables`
   - If freshness SLAs are critical: consider setting up Snowflake Alerts or dbt source freshness tests
