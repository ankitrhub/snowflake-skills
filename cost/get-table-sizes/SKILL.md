---
name: get-table-sizes
description: Find the largest tables by storage size (active, time-travel, fail-safe) using TABLE_STORAGE_METRICS.
---

# Get Table Sizes

## Goal
Quickly answer:
- Which tables consume the **most storage**?
- How much storage is in **active data** vs **time travel** vs **fail-safe** vs **retained for clones**?
- Which **dropped tables** are still incurring storage costs?
- What is the **total billed storage** for a database or schema?

## Inputs to collect
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `table_name`
- Optional: `include_dropped` (default false)
- Optional: `limit` (default 50)

## Workflow

### 1) Top tables by total storage (Account Usage)

> **Note:** `ACCOUNT_USAGE.TABLE_STORAGE_METRICS` has up to **90 minutes** latency.
> It covers all tables (including dropped ones still incurring costs).
> Requires the **ACCOUNTADMIN** role or equivalent privileges.
> Storage metrics for hybrid tables are **not** tracked in this view.

```sql
SELECT
  table_catalog  AS database_name,
  table_schema   AS schema_name,
  table_name,
  is_transient,
  ACTIVE_BYTES / POWER(1024, 3)                AS active_gb,
  TIME_TRAVEL_BYTES / POWER(1024, 3)           AS time_travel_gb,
  FAILSAFE_BYTES / POWER(1024, 3)              AS failsafe_gb,
  RETAINED_FOR_CLONE_BYTES / POWER(1024, 3)    AS retained_for_clone_gb,
  (ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES + RETAINED_FOR_CLONE_BYTES)
    / POWER(1024, 3)                           AS total_billed_gb,
  table_created,
  table_dropped,
  comment
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
  AND (<INCLUDE_DROPPED> OR table_dropped IS NULL)
ORDER BY total_billed_gb DESC
LIMIT <LIMIT>;
```

### 2) Storage breakdown by database
Summarize storage at the database level to find which databases consume the most.

```sql
SELECT
  table_catalog AS database_name,
  COUNT(*)      AS table_count,
  SUM(ACTIVE_BYTES) / POWER(1024, 4)                AS active_tb,
  SUM(TIME_TRAVEL_BYTES) / POWER(1024, 4)           AS time_travel_tb,
  SUM(FAILSAFE_BYTES) / POWER(1024, 4)              AS failsafe_tb,
  SUM(RETAINED_FOR_CLONE_BYTES) / POWER(1024, 4)    AS retained_for_clone_tb,
  SUM(ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES + RETAINED_FOR_CLONE_BYTES)
    / POWER(1024, 4)                                 AS total_billed_tb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE (<INCLUDE_DROPPED> OR table_dropped IS NULL)
GROUP BY table_catalog
ORDER BY total_billed_tb DESC;
```

### 3) Storage breakdown by schema (within a database)
Drill into a specific database to find expensive schemas.

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  COUNT(*)      AS table_count,
  SUM(ACTIVE_BYTES) / POWER(1024, 3)                AS active_gb,
  SUM(TIME_TRAVEL_BYTES) / POWER(1024, 3)           AS time_travel_gb,
  SUM(FAILSAFE_BYTES) / POWER(1024, 3)              AS failsafe_gb,
  SUM(RETAINED_FOR_CLONE_BYTES) / POWER(1024, 3)    AS retained_for_clone_gb,
  SUM(ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES + RETAINED_FOR_CLONE_BYTES)
    / POWER(1024, 3)                                 AS total_billed_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE table_catalog = '<DATABASE_NAME>'
  AND (<INCLUDE_DROPPED> OR table_dropped IS NULL)
GROUP BY table_catalog, table_schema
ORDER BY total_billed_gb DESC;
```

### 4) Dropped tables still incurring costs
Find tables that were dropped but are still billing for time-travel, fail-safe, or clone retention.

```sql
SELECT
  table_catalog  AS database_name,
  table_schema   AS schema_name,
  table_name,
  is_transient,
  table_dropped,
  table_entered_failsafe,
  ACTIVE_BYTES / POWER(1024, 3)                AS active_gb,
  TIME_TRAVEL_BYTES / POWER(1024, 3)           AS time_travel_gb,
  FAILSAFE_BYTES / POWER(1024, 3)              AS failsafe_gb,
  RETAINED_FOR_CLONE_BYTES / POWER(1024, 3)    AS retained_for_clone_gb,
  (ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES + RETAINED_FOR_CLONE_BYTES)
    / POWER(1024, 3)                           AS total_billed_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE table_dropped IS NOT NULL
  AND (ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES + RETAINED_FOR_CLONE_BYTES) > 0
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
ORDER BY total_billed_gb DESC
LIMIT <LIMIT>;
```

### 5) (Optional) Fresh per-table sizes via Information Schema
If you need real-time sizes without Account Usage latency (but still requires ACCOUNTADMIN):

```sql
SELECT
  table_catalog  AS database_name,
  table_schema   AS schema_name,
  table_name,
  ACTIVE_BYTES / POWER(1024, 3)                AS active_gb,
  TIME_TRAVEL_BYTES / POWER(1024, 3)           AS time_travel_gb,
  FAILSAFE_BYTES / POWER(1024, 3)              AS failsafe_gb,
  RETAINED_FOR_CLONE_BYTES / POWER(1024, 3)    AS retained_for_clone_gb,
  (ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES + RETAINED_FOR_CLONE_BYTES)
    / POWER(1024, 3)                           AS total_billed_gb
FROM <DATABASE_NAME>.INFORMATION_SCHEMA.TABLE_STORAGE_METRICS
WHERE (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
  AND (<INCLUDE_DROPPED> OR table_dropped IS NULL)
ORDER BY total_billed_gb DESC
LIMIT <LIMIT>;
```

> **Note:** Information Schema `TABLE_STORAGE_METRICS` still has 1-2 hour delay for byte columns.
> It is scoped to a single database and also requires ACCOUNTADMIN.

### 6) (Optional) Quick row count + byte size via Information Schema (SELECT-only)
For a fast, privilege-friendly row count estimate without ACCOUNTADMIN:

```sql
SELECT
  table_catalog AS database_name,
  table_schema AS schema_name,
  table_name,
  table_type AS table_kind,
  row_count,
  bytes,
  bytes / POWER(1024, 3) AS size_gb,
  created AS created_on
FROM <DATABASE_NAME>.INFORMATION_SCHEMA.TABLES
WHERE table_schema = '<SCHEMA_NAME>'
ORDER BY bytes DESC;
```

Optional fallback (if your environment allows `SHOW` commands):

```sql
SHOW TABLES IN SCHEMA <DATABASE_NAME>.<SCHEMA_NAME>;
```

```sql
SELECT
  "database_name" AS database_name,
  "schema_name" AS schema_name,
  "name" AS table_name,
  "kind" AS table_kind,
  "rows" AS row_count,
  "bytes" AS bytes,
  "bytes" / POWER(1024, 3) AS size_gb,
  "created_on" AS created_on
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
ORDER BY "bytes" DESC;
```

> **Note:** Row counts and bytes in these quick metadata paths are estimates updated periodically by Snowflake.
> They do not include time-travel, fail-safe, or clone-retained bytes.

## Output format
Return:
1) Top tables by total billed storage (ranked table with GB/TB breakdown)
2) Storage breakdown by category (active vs time-travel vs fail-safe vs retained-for-clone)
3) Key observations:
   - Tables where time-travel or fail-safe bytes dominate (consider reducing DATA_RETENTION_TIME_IN_DAYS or using transient tables)
   - Dropped tables still incurring costs
   - Clone groups sharing storage (same CLONE_GROUP_ID)
4) Recommended next steps:
   - Reduce retention period for large tables where long time-travel is unnecessary
   - Drop unused tables that are still in time-travel
   - Consider transient tables for staging/ETL scratch tables (no fail-safe)
   - Use `table-dml-activity` to correlate storage growth with DML patterns
