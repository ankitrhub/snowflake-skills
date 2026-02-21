---
name: pipeline-run-health
description: Monitor full task graph (DAG) pipeline runs -- end-to-end duration, bottleneck tasks, failure patterns, and run history.
---

# Pipeline Run Health (Task Graphs)

## Goal
Quickly answer:
- How long does my **full pipeline (DAG) take** end-to-end?
- Which **task is the bottleneck** in the graph?
- Are pipeline runs **succeeding, failing, or getting slower** over time?
- What was the **first error** in a failed pipeline run?

## Inputs to collect
- `database_name` (required)
- `schema_name` (required)
- Optional: `root_task_name` (if omitted, show all DAGs in the schema)
- `lookback_days` (optional, default 7)
- Optional: `limit` (optional, default 50)

## Workflow

### 1) Recent pipeline (DAG) runs -- overview

> **Note:** `COMPLETE_TASK_GRAPHS` has up to **45 minutes** latency.
> It shows one row per completed DAG run (not per individual task).
> `STATE` values: `SUCCEEDED`, `FAILED`, `CANCELLED`.
> Runs where the root task was `SKIPPED` are not included.

```sql
SELECT
  root_task_name,
  database_name,
  schema_name,
  state,
  scheduled_from,
  scheduled_time,
  query_start_time,
  completed_time,
  DATEDIFF('second', scheduled_time, completed_time) AS total_duration_sec,
  ROUND(DATEDIFF('second', scheduled_time, completed_time) / 60.0, 1) AS total_duration_min,
  first_error_task_name,
  first_error_code,
  first_error_message,
  attempt_number,
  graph_version,
  graph_run_group_id
FROM SNOWFLAKE.ACCOUNT_USAGE.COMPLETE_TASK_GRAPHS
WHERE completed_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
  AND (<ROOT_TASK_NAME_IS_NULL> OR root_task_name = '<ROOT_TASK_NAME>')
ORDER BY scheduled_time DESC
LIMIT <LIMIT>;
```

### 2) Pipeline success/failure summary
Aggregated view of DAG run health over the lookback window.

```sql
SELECT
  root_task_name,
  COUNT(*) AS total_runs,
  SUM(CASE WHEN state = 'SUCCEEDED' THEN 1 ELSE 0 END) AS succeeded,
  SUM(CASE WHEN state = 'FAILED' THEN 1 ELSE 0 END) AS failed,
  SUM(CASE WHEN state = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled,
  ROUND(SUM(CASE WHEN state = 'SUCCEEDED' THEN 1 ELSE 0 END)::FLOAT / NULLIF(COUNT(*), 0) * 100, 1) AS success_rate_pct,
  AVG(DATEDIFF('second', scheduled_time, completed_time)) / 60.0 AS avg_duration_min,
  MAX(DATEDIFF('second', scheduled_time, completed_time)) / 60.0 AS max_duration_min,
  MIN(DATEDIFF('second', scheduled_time, completed_time)) / 60.0 AS min_duration_min
FROM SNOWFLAKE.ACCOUNT_USAGE.COMPLETE_TASK_GRAPHS
WHERE completed_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
  AND (<ROOT_TASK_NAME_IS_NULL> OR root_task_name = '<ROOT_TASK_NAME>')
GROUP BY root_task_name
ORDER BY failed DESC, total_runs DESC;
```

### 3) Per-task breakdown for a specific DAG run
Drill into a single pipeline run to find the bottleneck task.

> **Note:** `TASK_HISTORY` (Account Usage) has up to **45 minutes** latency and covers 365 days.
> Use `GRAPH_RUN_GROUP_ID` + `ATTEMPT_NUMBER` to uniquely identify a DAG run.
> `STATE` in this view: `SUCCEEDED`, `FAILED`, `CANCELLED`, `SKIPPED`.

```sql
SELECT
  name AS task_name,
  state,
  scheduled_time,
  query_start_time,
  completed_time,
  DATEDIFF('second', scheduled_time, completed_time) AS task_duration_sec,
  DATEDIFF('second', query_start_time, completed_time) AS query_duration_sec,
  DATEDIFF('second', scheduled_time, query_start_time) AS wait_time_sec,
  query_id,
  error_code,
  error_message,
  attempt_number,
  condition_text
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE graph_run_group_id = '<GRAPH_RUN_GROUP_ID>'
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
ORDER BY scheduled_time;
```

### 4) Find the bottleneck task across multiple runs
Identify which task consistently takes the longest within a DAG.

```sql
SELECT
  th.name AS task_name,
  COUNT(*) AS runs,
  AVG(DATEDIFF('second', th.query_start_time, th.completed_time)) AS avg_query_sec,
  MAX(DATEDIFF('second', th.query_start_time, th.completed_time)) AS max_query_sec,
  AVG(DATEDIFF('second', th.scheduled_time, th.query_start_time)) AS avg_wait_sec,
  SUM(CASE WHEN th.state = 'FAILED' THEN 1 ELSE 0 END) AS failure_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY th
WHERE th.completed_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND th.database_name = '<DATABASE_NAME>'
  AND th.schema_name = '<SCHEMA_NAME>'
  AND th.root_task_id = (
    SELECT "id" FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
    WHERE "name" = '<ROOT_TASK_NAME>'
    LIMIT 1
  )
GROUP BY th.name
ORDER BY avg_query_sec DESC;
```

**Alternative** (if you already know the ROOT_TASK_ID):
```sql
SELECT
  name AS task_name,
  COUNT(*) AS runs,
  AVG(DATEDIFF('second', query_start_time, completed_time)) AS avg_query_sec,
  MAX(DATEDIFF('second', query_start_time, completed_time)) AS max_query_sec,
  AVG(DATEDIFF('second', scheduled_time, query_start_time)) AS avg_wait_sec,
  SUM(CASE WHEN state = 'FAILED' THEN 1 ELSE 0 END) AS failure_count
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE completed_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
  AND root_task_id = '<ROOT_TASK_ID>'
GROUP BY name
ORDER BY avg_query_sec DESC;
```

### 5) Pipeline duration trend (is the DAG getting slower?)

```sql
SELECT
  DATE_TRUNC('day', scheduled_time) AS run_date,
  root_task_name,
  COUNT(*) AS runs,
  AVG(DATEDIFF('second', scheduled_time, completed_time)) / 60.0 AS avg_duration_min,
  MAX(DATEDIFF('second', scheduled_time, completed_time)) / 60.0 AS max_duration_min,
  SUM(CASE WHEN state = 'FAILED' THEN 1 ELSE 0 END) AS failures
FROM SNOWFLAKE.ACCOUNT_USAGE.COMPLETE_TASK_GRAPHS
WHERE completed_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
  AND (<ROOT_TASK_NAME_IS_NULL> OR root_task_name = '<ROOT_TASK_NAME>')
GROUP BY run_date, root_task_name
ORDER BY run_date DESC;
```

### 6) Most common failure errors across DAG runs

```sql
SELECT
  first_error_task_name,
  first_error_code,
  first_error_message,
  COUNT(*) AS occurrence_count,
  MIN(scheduled_time) AS first_seen,
  MAX(scheduled_time) AS last_seen
FROM SNOWFLAKE.ACCOUNT_USAGE.COMPLETE_TASK_GRAPHS
WHERE completed_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND state = 'FAILED'
  AND database_name = '<DATABASE_NAME>'
  AND schema_name = '<SCHEMA_NAME>'
  AND (<ROOT_TASK_NAME_IS_NULL> OR root_task_name = '<ROOT_TASK_NAME>')
GROUP BY first_error_task_name, first_error_code, first_error_message
ORDER BY occurrence_count DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) Recent DAG runs with status and duration
2) Success rate and duration summary per DAG
3) Per-task breakdown for a specific run (showing the bottleneck)
4) Duration trend (is the pipeline getting slower?)
5) Top failure errors grouped by task and message
6) Recommendations:
   - If one task dominates duration: run `optimize-query-by-id` on its `query_id`
   - If wait_time_sec is high for child tasks: predecessors are the bottleneck, not the child itself
   - If success rate is low with same error: fix the root cause in the failing task first
   - If duration is trending up: correlate with `table-dml-activity` to check for growing data volumes
   - If SKIPPED tasks appear frequently: check WHEN conditions and stream health with `stream-has-data-and-staleness`
