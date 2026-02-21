---
name: monitor-task-failures
description: Monitor Snowflake task failures and recent execution history (INFORMATION_SCHEMA first, ACCOUNT_USAGE fallback).
---

# Monitor Task Failures

## Goal
Quickly answer:
- Which tasks failed recently?
- What was the error message / error code?
- What was the query_id for the failing run?
- When did each task last run, and what was its latest state?

## Inputs to collect
- `database_name` (required)
- `schema_name` (required)
- `task_name` (optional)
- `lookback_hours` (optional, default 24)
- `limit` (optional, default 200)

## Workflow

### 1) Recent failures (fresh, preferred)
Use the database-scoped Information Schema table function for near-real-time results:

> **Notes:**
> - The Info Schema function covers the **last 7 days** only.
> - `RESULT_LIMIT` defaults to **100** (max 10,000) and is applied **before** any WHERE/LIMIT clause.
> - Use `ERROR_ONLY => TRUE` to return only FAILED/CANCELLED runs directly.
> - If `TASK_NAME` is known, pass it as a parameter for more efficient filtering.

```sql
SELECT
  name,
  database_name,
  schema_name,
  state,
  scheduled_time,
  completed_time,
  return_value,
  query_id,
  error_code,
  error_message
FROM TABLE(<DATABASE_NAME>.INFORMATION_SCHEMA.TASK_HISTORY(
  SCHEDULED_TIME_RANGE_START => DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP()),
  ERROR_ONLY => TRUE,
  RESULT_LIMIT => 10000
))
WHERE schema_name = '<SCHEMA_NAME>'
  AND (<TASK_NAME_IS_NULL> OR name = '<TASK_NAME>')
ORDER BY scheduled_time DESC
LIMIT <LIMIT>;
```

### 2) Last run status per task (fresh)
```sql
WITH h AS (
  SELECT
    name,
    database_name,
    schema_name,
    state,
    scheduled_time,
    completed_time,
    query_id,
    error_code,
    error_message
  FROM TABLE(<DATABASE_NAME>.INFORMATION_SCHEMA.TASK_HISTORY(
    SCHEDULED_TIME_RANGE_START => DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP()),
    RESULT_LIMIT => 10000
  ))
  WHERE schema_name = '<SCHEMA_NAME>'
    AND (<TASK_NAME_IS_NULL> OR name = '<TASK_NAME>')
)
SELECT *
FROM h
QUALIFY ROW_NUMBER() OVER (PARTITION BY name ORDER BY scheduled_time DESC) = 1
ORDER BY state DESC, scheduled_time DESC;
```

### 3) Fallback (older / delayed): ACCOUNT_USAGE.TASK_HISTORY
If Information Schema doesn’t return rows (older history), fall back:

```sql
SELECT
  name,
  database_name,
  schema_name,
  state,
  scheduled_time,
  completed_time,
  query_id,
  error_code,
  error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
  AND state = 'FAILED'
  AND (<TASK_NAME_IS_NULL> OR name = '<TASK_NAME>')
ORDER BY scheduled_time DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) A table of recent failures (task, state, scheduled_time, error_message, query_id)
2) A “latest run per task” table
3) Short recommendations:
   - If many failures share a single `error_message`, prioritize fixing that root cause.
   - Use `optimize-query-by-id` on the failing `query_id` if performance-related.
