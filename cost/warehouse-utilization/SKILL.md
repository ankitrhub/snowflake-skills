---
name: warehouse-utilization
description: Analyze warehouse credit spend, idle time, workload, and queueing to right-size warehouses.
---

# Warehouse Utilization

## Goal
Quickly answer:
- Which warehouses consume the **most credits**?
- How much credit spend is **idle time** vs actual query execution?
- Are warehouses **overloaded** (queueing) or **under-utilized** (consistently low load)?
- What does the **hourly workload profile** look like for a specific warehouse?

## Inputs to collect
- `lookback_days` (optional, default 7)
- Optional: `warehouse_name`
- Optional: `limit` (optional, default 20)

## Workflow

### 1) Credit spend per warehouse (ranked)

> **Note:** `WAREHOUSE_METERING_HISTORY` has up to **3 hours** latency (up to **6 hours** for `CREDITS_USED_CLOUD_SERVICES`).
> `CREDITS_USED` is the sum of compute + cloud services but does **not** reflect the cloud services billing adjustment.
> For actual billed amounts, use `METERING_DAILY_HISTORY`.

```sql
SELECT
  warehouse_name,
  SUM(credits_used)                         AS total_credits,
  SUM(credits_used_compute)                 AS total_compute_credits,
  SUM(credits_used_cloud_services)          AS total_cloud_services_credits,
  SUM(credits_attributed_compute_queries)   AS total_query_credits,
  SUM(credits_used_compute)
    - SUM(credits_attributed_compute_queries) AS total_idle_credits,
  ROUND(
    (SUM(credits_used_compute) - SUM(credits_attributed_compute_queries))
    / NULLIF(SUM(credits_used_compute), 0) * 100, 1
  ) AS idle_pct,
  COUNT(*) AS active_hours
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
GROUP BY warehouse_name
ORDER BY total_credits DESC
LIMIT <LIMIT>;
```

### 2) Daily credit trend per warehouse
Useful for spotting spend spikes or day-of-week patterns.

```sql
SELECT
  DATE_TRUNC('day', start_time) AS usage_date,
  warehouse_name,
  SUM(credits_used)                       AS daily_credits,
  SUM(credits_used_compute)               AS daily_compute_credits,
  SUM(credits_attributed_compute_queries) AS daily_query_credits,
  SUM(credits_used_compute)
    - SUM(credits_attributed_compute_queries) AS daily_idle_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
GROUP BY usage_date, warehouse_name
ORDER BY usage_date DESC, daily_credits DESC;
```

### 3) Warehouse workload analysis (queueing, blocking, utilization)

> **Note:** `WAREHOUSE_LOAD_HISTORY` shows data in **5-minute intervals** with timestamps in **UTC**.
> `AVG_RUNNING` is a ratio: e.g. 1.0 means all execution time was used; >1.0 means multiple concurrent queries.
> `AVG_QUEUED_LOAD` > 0 consistently signals the warehouse is overloaded and may need scaling up.

```sql
SELECT
  warehouse_name,
  AVG(avg_running)              AS avg_running_load,
  MAX(avg_running)              AS peak_running_load,
  AVG(avg_queued_load)          AS avg_queued_overload,
  MAX(avg_queued_load)          AS peak_queued_overload,
  AVG(avg_queued_provisioning)  AS avg_queued_provisioning,
  AVG(avg_blocked)              AS avg_blocked_load,
  COUNT(*) AS intervals_5min
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
GROUP BY warehouse_name
ORDER BY avg_queued_overload DESC;
```

### 4) Hourly heatmap for a specific warehouse
Identify busy hours and idle hours to inform auto-suspend or scheduling decisions.

```sql
SELECT
  EXTRACT(DOW FROM start_time) AS day_of_week,
  EXTRACT(HOUR FROM start_time) AS hour_of_day,
  AVG(avg_running)              AS avg_running_load,
  AVG(avg_queued_load)          AS avg_queued_overload,
  SUM(credits_used)             AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY l
JOIN SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY m
  ON l.warehouse_id = m.warehouse_id
 AND DATE_TRUNC('hour', l.start_time) = m.start_time
WHERE l.start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND l.warehouse_name = '<WAREHOUSE_NAME>'
GROUP BY day_of_week, hour_of_day
ORDER BY day_of_week, hour_of_day;
```

### 5) Identify warehouses with high idle cost

```sql
SELECT
  warehouse_name,
  SUM(credits_used_compute) AS compute_credits,
  SUM(credits_attributed_compute_queries) AS query_credits,
  SUM(credits_used_compute) - SUM(credits_attributed_compute_queries) AS idle_credits,
  ROUND(
    (SUM(credits_used_compute) - SUM(credits_attributed_compute_queries))
    / NULLIF(SUM(credits_used_compute), 0) * 100, 1
  ) AS idle_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
HAVING idle_credits > 0
ORDER BY idle_credits DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) Ranked warehouses by total credit spend
2) Idle time analysis (idle credits and idle %)
3) Workload signal: queueing (overloaded) vs low running load (under-utilized)
4) Recommendations:
   - High idle %: reduce auto-suspend timeout, or switch to a smaller warehouse
   - High queued load: enable multi-cluster warehouse or scale up size
   - High blocked load: investigate concurrent DML / transaction locks
   - Consistent low running load: downsize the warehouse
