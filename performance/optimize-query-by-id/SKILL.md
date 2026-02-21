---
name: optimize-query-by-id
description: Diagnose and optimize a Snowflake query when the user provides a query_id.
---

# Optimize Query from Query ID

Goal: **Fetch query details → fetch operator stats (if available) → identify bottlenecks → recommend changes → optionally propose an optimized SQL rewrite**.

## Inputs to collect
- `query_id` (required)
- Optional: `include_operator_stats` (default true)
- Optional: `include_query_insights` (default true; requires ACCOUNT_USAGE.QUERY_INSIGHTS access)

## Workflow

### 1) Fetch query runtime + scan metrics
Try **Information Schema** first (fresh / last ~7 days):

> **Important:** `QUERY_HISTORY()` without `RESULT_LIMIT` returns only the last **100** queries
> for the current user. Always pass `RESULT_LIMIT => 10000` to maximize the search window.
> This function only covers the **last 7 days**; fall back to Account Usage for older queries.

```sql
SELECT
  query_id,
  query_text,
  database_name,
  schema_name,
  warehouse_name,
  user_name,
  role_name,
  query_hash,
  query_parameterized_hash,
  start_time,
  end_time,
  total_elapsed_time/1000 AS seconds,
  compilation_time/1000 AS compilation_seconds,
  execution_time/1000 AS execution_seconds,
  queued_overload_time/1000 AS queued_overload_seconds,
  queued_provisioning_time/1000 AS queued_provisioning_seconds,
  queued_repair_time/1000 AS queued_repair_seconds,
  transaction_blocked_time/1000 AS blocked_seconds,
  bytes_scanned/1e9 AS gb_scanned,
  percentage_scanned_from_cache AS cache_ratio,
  partitions_scanned,
  partitions_total,
  bytes_spilled_to_local_storage/1e9 AS gb_spilled_local,
  bytes_spilled_to_remote_storage/1e9 AS gb_spilled_remote,
  bytes_sent_over_the_network/1e9 AS gb_sent_over_network,
  rows_produced
FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY(
  RESULT_LIMIT => 10000
))
WHERE query_id = '<QUERY_ID>';
```

If no row is returned (older than 7 days), fall back to **Account Usage** (up to 365 days; can be delayed up to ~45 min):

```sql
SELECT
  query_id,
  query_text,
  database_name,
  schema_name,
  warehouse_name,
  user_name,
  role_name,
  query_hash,
  query_parameterized_hash,
  start_time,
  end_time,
  total_elapsed_time/1000 AS seconds,
  compilation_time/1000 AS compilation_seconds,
  execution_time/1000 AS execution_seconds,
  queued_overload_time/1000 AS queued_overload_seconds,
  queued_provisioning_time/1000 AS queued_provisioning_seconds,
  queued_repair_time/1000 AS queued_repair_seconds,
  transaction_blocked_time/1000 AS blocked_seconds,
  bytes_scanned/1e9 AS gb_scanned,
  percentage_scanned_from_cache AS cache_ratio,
  partitions_scanned,
  partitions_total,
  bytes_spilled_to_local_storage/1e9 AS gb_spilled_local,
  bytes_spilled_to_remote_storage/1e9 AS gb_spilled_remote,
  bytes_sent_over_the_network/1e9 AS gb_sent_over_network,
  rows_produced
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<QUERY_ID>';
```

### 2) (Optional) Fetch query insights (plain-English recommendations)
If available:

```sql
SELECT
  query_id,
  insight_type_id,
  insight_topic,
  message,
  suggestions,
  is_opportunity
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE query_id = '<QUERY_ID>'
ORDER BY insight_topic, insight_type_id;
```

### 3) (Optional) Fetch operator-level stats
If the account supports query profiling for this query:

```sql
SELECT *
FROM TABLE(GET_QUERY_OPERATOR_STATS('<QUERY_ID>'));
```

Interpretation hints:
- Look for **TableScan** operators with high bytes or no pruning.
- Look for **Join explosions** (output rows ≫ input rows).
- Look for **Sort/Aggregate** operators with spillage.

### 4) Diagnose and categorize bottlenecks
Use these common heuristics:
- **Queueing**: `queued_overload_time` is large ⇒ warehouse overloaded / concurrency.
- **Blocking**: `transaction_blocked_time` is large ⇒ lock contention / concurrent DML.
- **No pruning**: `partitions_scanned = partitions_total` (or close) ⇒ filters not selective / misaligned with clustering.
- **Spillage**: spilled bytes > 0 ⇒ memory pressure (joins/aggregations); consider query rewrite and/or warehouse sizing.
- **Scan inefficiency**: `gb_scanned` high relative to rows produced ⇒ reduce scanned columns / push down filters.

### 5) Recommend improvements (safe + actionable)
Provide:
1) A short bullet summary of the top 1–3 bottlenecks.
2) A small metrics table (seconds, GB scanned, partitions scanned/total, spillage).
3) Specific recommended actions.

### 6) (Optional) Propose an optimized SQL rewrite
Only rewrite SQL when you can preserve semantics. Prefer conservative improvements:
- replace `SELECT *` with required columns
- move selective filters earlier (CTEs)
- avoid functions on filter columns when equivalent range predicates exist
- fix clearly incorrect joins (missing join condition)

If you cannot guarantee identical results, **do not rewrite**.

## Output format
Return:
1) **Diagnosis summary** (plain English)
2) **Key metrics** (table)
3) **Evidence** (which columns/values triggered the diagnosis)
4) **Recommended next steps**
5) (Optional) **Optimized SQL**
