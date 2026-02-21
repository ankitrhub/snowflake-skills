---
name: monitor-dynamic-table-refreshes
description: Monitor dynamic table refresh health, failures, lag, and performance using DYNAMIC_TABLE_REFRESH_HISTORY.
---

# Monitor Dynamic Table Refreshes

## Goal
Quickly answer:
- Which dynamic tables have **failed or cancelled** refreshes recently?
- Are dynamic tables **meeting their target lag** (completion target)?
- How long do refreshes take, and are they **getting slower** over time?
- What is the **last successful refresh** per dynamic table?

## Inputs to collect
- `lookback_days` (optional, default 7)
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `table_name` (the dynamic table name)
- Optional: `limit` (optional, default 100)

## Workflow

### 1) Failed and cancelled refreshes

> **Note:** `DYNAMIC_TABLE_REFRESH_HISTORY` has up to **3 hours** latency.
> Requires `SNOWFLAKE.USAGE_VIEWER` database role or MONITOR privilege on the dynamic tables.
> `STATE` values: `EXECUTING`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `UPSTREAM_FAILED`.

```sql
SELECT
  database_name,
  schema_name,
  name AS dynamic_table_name,
  state,
  state_code,
  state_message,
  query_id,
  data_timestamp,
  refresh_start_time,
  refresh_end_time,
  refresh_action,
  refresh_trigger,
  target_lag_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY
WHERE data_timestamp >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND state IN ('FAILED', 'CANCELLED', 'UPSTREAM_FAILED')
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR name = '<TABLE_NAME>')
ORDER BY data_timestamp DESC
LIMIT <LIMIT>;
```

### 2) Lag analysis: refreshes that missed their completion target

```sql
SELECT
  database_name,
  schema_name,
  name AS dynamic_table_name,
  data_timestamp,
  refresh_start_time,
  refresh_end_time,
  completion_target,
  DATEDIFF('second', refresh_start_time, refresh_end_time) AS refresh_duration_sec,
  CASE
    WHEN refresh_end_time > completion_target THEN 'MISSED'
    ELSE 'MET'
  END AS target_status,
  DATEDIFF('second', completion_target, refresh_end_time) AS seconds_over_target,
  target_lag_sec,
  refresh_action,
  refresh_trigger
FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY
WHERE data_timestamp >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND state = 'SUCCEEDED'
  AND refresh_end_time > completion_target
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR name = '<TABLE_NAME>')
ORDER BY seconds_over_target DESC
LIMIT <LIMIT>;
```

### 3) Last successful refresh per dynamic table

```sql
SELECT
  database_name,
  schema_name,
  name AS dynamic_table_name,
  state,
  data_timestamp,
  refresh_start_time,
  refresh_end_time,
  DATEDIFF('second', refresh_start_time, refresh_end_time) AS refresh_duration_sec,
  refresh_action,
  target_lag_sec,
  statistics:numInsertedRows::number AS rows_inserted,
  statistics:numDeletedRows::number AS rows_deleted,
  statistics:numCopiedRows::number AS rows_copied
FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY
WHERE data_timestamp >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND state = 'SUCCEEDED'
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR name = '<TABLE_NAME>')
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY database_name, schema_name, name
  ORDER BY data_timestamp DESC
) = 1
ORDER BY dynamic_table_name;
```

### 4) Refresh duration trend (is a DT getting slower?)

```sql
SELECT
  DATE_TRUNC('day', data_timestamp) AS refresh_date,
  database_name,
  schema_name,
  name AS dynamic_table_name,
  COUNT(*) AS refresh_count,
  AVG(DATEDIFF('second', refresh_start_time, refresh_end_time)) AS avg_duration_sec,
  MAX(DATEDIFF('second', refresh_start_time, refresh_end_time)) AS max_duration_sec,
  SUM(CASE WHEN state = 'FAILED' THEN 1 ELSE 0 END) AS failures,
  SUM(CASE WHEN refresh_end_time > completion_target THEN 1 ELSE 0 END) AS missed_targets,
  AVG(statistics:numInsertedRows::number) AS avg_rows_inserted
FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY
WHERE data_timestamp >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND state IN ('SUCCEEDED', 'FAILED')
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR name = '<TABLE_NAME>')
GROUP BY refresh_date, database_name, schema_name, name
ORDER BY refresh_date DESC, dynamic_table_name;
```

### 5) Refresh type breakdown (incremental vs full)

```sql
SELECT
  database_name,
  schema_name,
  name AS dynamic_table_name,
  refresh_action,
  COUNT(*) AS refresh_count,
  AVG(DATEDIFF('second', refresh_start_time, refresh_end_time)) AS avg_duration_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.DYNAMIC_TABLE_REFRESH_HISTORY
WHERE data_timestamp >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND state = 'SUCCEEDED'
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR name = '<TABLE_NAME>')
GROUP BY database_name, schema_name, name, refresh_action
ORDER BY dynamic_table_name, refresh_count DESC;
```

## Output format
Return:
1) Failed/cancelled refreshes with error details and query_id
2) Lag violations (missed completion targets)
3) Last successful refresh per dynamic table
4) Duration trend (getting slower or stable)
5) Recommendations:
   - If UPSTREAM_FAILED: fix the upstream dynamic table first
   - If consistently missing target lag: increase target_lag or optimize the DT query
   - If FULL refreshes dominate: check dynamic table metadata for `refresh_mode` and `refresh_mode_reason` (prefer `INFORMATION_SCHEMA.DYNAMIC_TABLES` / `ACCOUNT_USAGE.DYNAMIC_TABLES`)
   - If duration is increasing: check DML volume upstream with `table-dml-activity`
   - Use `optimize-query-by-id` on the failing `query_id` for detailed analysis
