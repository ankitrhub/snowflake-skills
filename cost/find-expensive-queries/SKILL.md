---
name: find-expensive-queries
description: Find and rank expensive Snowflake queries by cost (credits), time, scan, or spill.
---

# Find Expensive Queries

## Inputs to collect
- `lookback_days` (default 7)
- `metric` (default `cost`) one of: `cost`, `time`, `scan`, `spill`
- Optional: `warehouse_name`
- Optional: `user_name`
- Optional: `query_tag`
- Optional: `limit` (default 20)

## Workflow

### 1) If metric = cost (preferred)
Use Account Usage attribution (if permitted).

> **Note:** `QUERY_ATTRIBUTION_HISTORY` has up to **8 hours** of latency.
> This view does **not** include `role_name`; join to `QUERY_HISTORY` for that.

```sql
SELECT
  query_id,
  warehouse_name,
  user_name,
  query_tag,
  query_hash,
  query_parameterized_hash,
  start_time,
  end_time,
  credits_attributed_compute,
  credits_used_query_acceleration
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
  AND (<USER_NAME_IS_NULL> OR user_name = '<USER_NAME>')
  AND (<QUERY_TAG_IS_NULL> OR query_tag = '<QUERY_TAG>')
ORDER BY credits_attributed_compute DESC
LIMIT <LIMIT>;
```

Then, pull query text + performance stats for those query_ids:

```sql
SELECT
  query_id,
  warehouse_name,
  user_name,
  query_tag,
  total_elapsed_time/1000 AS seconds,
  bytes_scanned/1e9 AS gb_scanned,
  bytes_spilled_to_local_storage/1e9 AS gb_spilled_local,
  bytes_spilled_to_remote_storage/1e9 AS gb_spilled_remote,
  partitions_scanned,
  partitions_total,
  queued_overload_time/1000 AS queued_overload_seconds,
  transaction_blocked_time/1000 AS blocked_seconds,
  LEFT(query_text, 500) AS query_text_preview
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id IN (<QUERY_ID_LIST>);
```

### 2) If metric != cost or attribution history not available
Rank from QUERY_HISTORY directly:

#### By time
```sql
SELECT
  query_id,
  start_time,
  warehouse_name,
  user_name,
  query_tag,
  total_elapsed_time/1000 AS seconds,
  bytes_scanned/1e9 AS gb_scanned,
  bytes_spilled_to_remote_storage/1e9 AS gb_spilled_remote,
  LEFT(query_text, 500) AS query_text_preview
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
  AND (<USER_NAME_IS_NULL> OR user_name = '<USER_NAME>')
  AND (<QUERY_TAG_IS_NULL> OR query_tag = '<QUERY_TAG>')
ORDER BY total_elapsed_time DESC
LIMIT <LIMIT>;
```

#### By scan
Order by `bytes_scanned`.

#### By spill
Order by `bytes_spilled_to_remote_storage + bytes_spilled_to_local_storage`.

### 3) Identify patterns & recommendations
From the ranked set, flag:
- repeated queries via `query_hash` / `query_parameterized_hash` (opportunity to cache/results reuse)
- no pruning (`partitions_scanned ≈ partitions_total`)
- spillage (remote spill is especially expensive)
- queueing (warehouse overloaded)

## Output format
Return:
1) Ranked table (top N) with the chosen metric
2) “What I see” patterns (2–5 bullets)
3) Top recommended actions (3–7 items)
4) Which query_id(s) to run `optimize-query-by-id` on next
