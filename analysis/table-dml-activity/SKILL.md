---
name: table-dml-activity
description: View recent DML activity (inserts, updates, deletes) on tables using SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY.
---

# Table DML Activity

## Goal
Quickly answer:
- Which tables had the **most DML activity** (rows added/removed/updated) recently?
- What is the **hourly DML pattern** for a specific table?
- Are there unexpected **spikes or drops** in DML volume?

## Inputs to collect
- `lookback_days` (optional, default 7)
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `table_name`
- Optional: `limit` (optional, default 50)

## Workflow

### 1) Top tables by total DML activity (aggregated)

> **Note:** `TABLE_DML_HISTORY` has up to **6 hours** of latency.
> It excludes DML on hybrid tables and background maintenance (auto-clustering, MV refresh, search optimization).
> It **includes** Snowpipe-initiated DML.

```sql
SELECT
  table_id,
  ANY_VALUE(database_name) AS database_name,
  ANY_VALUE(schema_name)   AS schema_name,
  ANY_VALUE(table_name)    AS table_name,
  SUM(rows_added)          AS total_rows_added,
  SUM(rows_removed)        AS total_rows_removed,
  SUM(rows_updated)        AS total_rows_updated,
  SUM(rows_added) + SUM(rows_removed) + SUM(rows_updated) AS total_dml_rows,
  MIN(start_time)          AS earliest_activity,
  MAX(end_time)            AS latest_activity,
  COUNT(*)                 AS hourly_buckets_with_activity
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY table_id
ORDER BY total_dml_rows DESC
LIMIT <LIMIT>;
```

### 2) Hourly DML trend for a specific table
Use this when a `table_name` is provided (or pick the top table from step 1).

```sql
SELECT
  start_time,
  end_time,
  database_name,
  schema_name,
  table_name,
  rows_added,
  rows_removed,
  rows_updated,
  rows_added + rows_removed + rows_updated AS total_dml_rows
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND database_name = '<DATABASE_NAME>'
  AND schema_name   = '<SCHEMA_NAME>'
  AND table_name    = '<TABLE_NAME>'
ORDER BY start_time DESC;
```

### 3) Daily summary (rolled up)
Useful for spotting day-over-day trends or weekend vs. weekday differences.

```sql
SELECT
  DATE_TRUNC('day', start_time) AS activity_date,
  database_name,
  schema_name,
  table_name,
  SUM(rows_added)   AS daily_rows_added,
  SUM(rows_removed) AS daily_rows_removed,
  SUM(rows_updated) AS daily_rows_updated,
  SUM(rows_added) + SUM(rows_removed) + SUM(rows_updated) AS daily_total_dml_rows
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY activity_date, database_name, schema_name, table_name
ORDER BY activity_date DESC, daily_total_dml_rows DESC
LIMIT <LIMIT>;
```

### 4) (Optional) Correlate with clustering / search optimization costs
Join with `AUTOMATIC_CLUSTERING_HISTORY` or `SEARCH_OPTIMIZATION_HISTORY` to see if DML spikes triggered expensive maintenance.

```sql
SELECT
  d.table_name,
  DATE_TRUNC('day', d.start_time) AS activity_date,
  SUM(d.rows_added + d.rows_removed + d.rows_updated) AS daily_dml_rows,
  SUM(c.credits_used) AS clustering_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY d
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY c
  ON c.table_name = d.table_name
 AND c.schema_name = d.schema_name
 AND c.database_name = d.database_name
 AND DATE_TRUNC('day', c.start_time) = DATE_TRUNC('day', d.start_time)
WHERE d.start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND d.database_name = '<DATABASE_NAME>'
  AND d.schema_name   = '<SCHEMA_NAME>'
  AND d.table_name    = '<TABLE_NAME>'
GROUP BY d.table_name, activity_date
ORDER BY activity_date DESC;
```

## Output format
Return:
1) Top tables by DML volume (ranked table)
2) Hourly or daily trend (if specific table requested)
3) Observations:
   - Tables with only inserts (append-only / streaming pattern)
   - Tables with high deletes (purge / CDC pattern)
   - Tables with high updates (SCD / merge pattern)
   - Unexpected zero-activity periods (pipeline stalls)
4) Recommended next steps:
   - If a table has sudden DML spikes, investigate with `find-expensive-queries` or `monitor-task-failures`
   - If clustering credits correlate with DML volume, consider adjusting clustering keys or batching writes
