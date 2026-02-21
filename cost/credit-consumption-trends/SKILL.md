---
name: credit-consumption-trends
description: Track daily credit consumption across all service types to spot anomalies and forecast spend.
---

# Credit Consumption Trends

## Goal
Quickly answer:
- What is the **daily/weekly credit trend** across the account?
- Which **service types** (warehouses, serverless tasks, pipes, auto-clustering, etc.) consume the most?
- Are there **unexpected spikes** or **week-over-week changes**?
- How much of cloud services was **actually billed** (after the 10% adjustment)?

## Inputs to collect
- `lookback_days` (optional, default 30)
- Optional: `service_type` (e.g. `WAREHOUSE_METERING`, `SERVERLESS_TASK`, `PIPE`, `AUTO_CLUSTERING`)

## Workflow

### 1) Daily credit trend (all services)

> **Note:** `METERING_DAILY_HISTORY` has up to **3 hours** latency.
> `CREDITS_BILLED` is the authoritative column for what was actually charged (includes the cloud services adjustment).
> Cloud services credits are only billed when daily consumption exceeds 10% of daily warehouse compute credits.

```sql
SELECT
  usage_date,
  SUM(credits_used_compute)                   AS compute_credits,
  SUM(credits_used_cloud_services)            AS cloud_services_credits,
  SUM(credits_adjustment_cloud_services)      AS cloud_services_adjustment,
  SUM(credits_billed)                         AS total_billed_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_DATE())
GROUP BY usage_date
ORDER BY usage_date DESC;
```

### 2) Credit breakdown by service type (ranked)
Identify which services are the biggest credit consumers.

```sql
SELECT
  service_type,
  SUM(credits_billed) AS total_billed_credits,
  SUM(credits_used_compute) AS compute_credits,
  SUM(credits_used_cloud_services) AS cloud_services_credits,
  ROUND(
    SUM(credits_billed) / NULLIF(SUM(SUM(credits_billed)) OVER (), 0) * 100, 1
  ) AS pct_of_total
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_DATE())
  AND (<SERVICE_TYPE_IS_NULL> OR service_type = '<SERVICE_TYPE>')
GROUP BY service_type
ORDER BY total_billed_credits DESC;
```

### 3) Daily trend for a specific service type

```sql
SELECT
  usage_date,
  service_type,
  credits_used_compute,
  credits_used_cloud_services,
  credits_adjustment_cloud_services,
  credits_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_DATE())
  AND service_type = '<SERVICE_TYPE>'
ORDER BY usage_date DESC;
```

### 4) Week-over-week comparison
Compare current week to previous week to detect anomalies.

```sql
WITH daily AS (
  SELECT
    usage_date,
    SUM(credits_billed) AS daily_billed
  FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
  WHERE usage_date >= DATEADD('day', -14, CURRENT_DATE())
  GROUP BY usage_date
),
weekly AS (
  SELECT
    CASE
      WHEN usage_date >= DATEADD('day', -7, CURRENT_DATE()) THEN 'current_week'
      ELSE 'prior_week'
    END AS week_label,
    SUM(daily_billed) AS week_total,
    AVG(daily_billed) AS daily_avg
  FROM daily
  GROUP BY week_label
)
SELECT
  w1.week_total AS current_week_total,
  w1.daily_avg  AS current_week_daily_avg,
  w2.week_total AS prior_week_total,
  w2.daily_avg  AS prior_week_daily_avg,
  ROUND((w1.week_total - w2.week_total) / NULLIF(w2.week_total, 0) * 100, 1) AS wow_change_pct
FROM weekly w1, weekly w2
WHERE w1.week_label = 'current_week'
  AND w2.week_label = 'prior_week';
```

### 5) Cloud services billing check
Show days where cloud services were actually billed (exceeded the 10% threshold).

```sql
SELECT
  usage_date,
  SUM(credits_used_cloud_services)        AS cloud_services_used,
  SUM(credits_adjustment_cloud_services)  AS cloud_services_adjustment,
  SUM(credits_used_cloud_services)
    + SUM(credits_adjustment_cloud_services) AS cloud_services_actually_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_DATE())
  AND credits_used_cloud_services > 0
GROUP BY usage_date
HAVING cloud_services_actually_billed > 0
ORDER BY cloud_services_actually_billed DESC;
```

## Output format
Return:
1) Daily credit trend (table or chart-friendly data)
2) Breakdown by service type (ranked, with % of total)
3) Week-over-week comparison with % change
4) Observations:
   - Fastest-growing service type
   - Days with unusual spikes
   - Cloud services billing events
5) Recommended next steps:
   - If WAREHOUSE_METERING dominates, run `warehouse-utilization` for right-sizing
   - If AUTO_CLUSTERING is high, run `table-dml-activity` to see if DML spikes trigger it
   - If SERVERLESS_TASK is high, run `monitor-task-failures` to check for retries
