---
name: monitor-long-running-queries
description: Identify currently-running long queries and provide queueing/blocking hints.
---

# Monitor Long-Running Queries

## Inputs to collect
- `min_running_minutes` (default 15)
- Optional: `warehouse_name`
- Optional: `lookback_minutes` (default 180) – how far back to search for “still running” queries
- Optional: `limit` (default 50)

## Workflow

### 1) Find currently running queries above threshold (fresh)
Use Information Schema (best effort):

```sql
WITH q AS (
  SELECT
    query_id,
    user_name,
    role_name,
    warehouse_name,
    query_tag,
    start_time,
    end_time,
    total_elapsed_time,
    bytes_scanned,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage,
    queued_overload_time,
    queued_provisioning_time,
    queued_repair_time,
    transaction_blocked_time,
    query_text
  FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
  WHERE start_time >= DATEADD('minute', -<LOOKBACK_MINUTES>, CURRENT_TIMESTAMP())
)
SELECT
  query_id,
  user_name,
  role_name,
  warehouse_name,
  query_tag,
  start_time,
  DATEDIFF('second', start_time, CURRENT_TIMESTAMP())/60.0 AS running_minutes,
  LEFT(query_text, 500) AS query_text_preview
FROM q
WHERE end_time IS NULL
  AND DATEDIFF('minute', start_time, CURRENT_TIMESTAMP()) >= <MIN_RUNNING_MINUTES>
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
ORDER BY start_time
LIMIT <LIMIT>;
```

### 2) Provide “likely cause” hints
For each running query, if queue/block/spill columns are populated, classify:
- **QUEUED**: queued_overload_time is significant
- **BLOCKED**: transaction_blocked_time is significant
- **SCAN/COMPUTE HEAVY**: very large bytes_scanned
- **MEMORY PRESSURE**: spillage columns non-zero

### 3) (Optional) Warehouse health snapshot (recent completed)
If you want a warehouse-level signal, pull recent completed queries and summarize queueing:

> **Note:** `ACCOUNT_USAGE.QUERY_HISTORY` can have **45–180 minutes** latency.
> For very short lookback windows, results may be stale or empty.
> Consider using a longer lookback (e.g. 360 minutes) for this aggregation.

```sql
SELECT
  warehouse_name,
  COUNT(*) AS query_count,
  AVG(total_elapsed_time)/1000 AS avg_seconds,
  AVG(queued_overload_time)/1000 AS avg_queued_overload_seconds,
  MAX(queued_overload_time)/1000 AS max_queued_overload_seconds
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('minute', -<LOOKBACK_MINUTES>, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
GROUP BY warehouse_name
ORDER BY avg_queued_overload_seconds DESC;
```

## Output format
Return:
1) A list/table of running queries (query_id, running_minutes, warehouse, user, preview)
2) A short diagnosis: “mostly queued” vs “mostly blocked” vs “mostly heavy scans”
3) Next-step suggestions:
   - run `optimize-query-by-id` for the worst offenders
   - consider warehouse scaling / multi-cluster if queueing dominates
   - investigate DML/transactions if blocking dominates
