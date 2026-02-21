---
name: query-insights
description: Find and act on Snowflake query performance insights and suggestions using ACCOUNT_USAGE.QUERY_INSIGHTS.
---

# Query Insights

Use the [QUERY_INSIGHTS](https://docs.snowflake.com/en/sql-reference/account-usage/query_insights) Account Usage view to discover performance insights and recommended actions for queries.

## Goal
Quickly answer:
- What **insights and suggestions** does Snowflake have for a specific query or query pattern?
- Which queries in a time window have **actionable opportunities** (`is_opportunity = true`)?
- What are the **most common insight types** (table scan, join, aggregate, warehouse) in my workload?
- Which **long-running** queries have insights?

## Inputs to collect
- Optional: `query_id` (for a single query)
- Optional: `query_parameterized_hash` (for all queries sharing the same parameterized pattern)
- Optional: `warehouse_name` or `warehouse_id`
- Optional: `lookback_days` (default 7)
- Optional: `insight_topic` — one of: `TABLE_SCAN`, `JOIN`, `AGGREGATION`, `UNION`, `WAREHOUSE`
- Optional: `opportunities_only` (default false) — when true, filter to `is_opportunity = true` (insights with improvement suggestions)
- Optional: `min_elapsed_ms` (e.g. 3600000 for 1 hour) — filter to queries at least this long
- Optional: `limit` (default 100)

## Workflow

### 1) Insights for a specific query

> **Note:** QUERY_INSIGHTS has up to **90 minutes** latency. Column `total_elapsed_time` is in **milliseconds**.

```sql
SELECT
  query_id,
  start_time,
  end_time,
  total_elapsed_time / 1000 AS elapsed_seconds,
  warehouse_name,
  insight_type_id,
  insight_topic,
  is_opportunity,
  message,
  suggestions
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE query_id = '<QUERY_ID>'
ORDER BY start_time DESC;
```

### 2) Insights for a query pattern (parameterized hash)

Find insights for all queries that share the same parameterized SQL (same shape, different literals).

```sql
SELECT
  query_id,
  start_time,
  total_elapsed_time / 1000 AS elapsed_seconds,
  warehouse_name,
  insight_type_id,
  insight_topic,
  is_opportunity,
  message,
  suggestions
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE query_parameterized_hash = '<QUERY_PARAMETERIZED_HASH>'
ORDER BY start_time DESC
LIMIT <LIMIT>;
```

### 3) Recent insights in a time window

Optionally filter by warehouse and minimum elapsed time (e.g. only slow queries).

```sql
SELECT
  query_id,
  query_parameterized_hash,
  start_time,
  total_elapsed_time / 1000 AS elapsed_seconds,
  warehouse_name,
  insight_type_id,
  insight_topic,
  is_opportunity,
  message,
  suggestions
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
  AND (<MIN_ELAPSED_MS_IS_NULL> OR total_elapsed_time >= <MIN_ELAPSED_MS>)
ORDER BY start_time DESC
LIMIT <LIMIT>;
```

### 4) Only actionable opportunities

Restrict to insights that include improvement suggestions (`is_opportunity = true`). Use this to prioritize fixes.

```sql
SELECT
  query_id,
  query_parameterized_hash,
  start_time,
  total_elapsed_time / 1000 AS elapsed_seconds,
  warehouse_name,
  insight_type_id,
  insight_topic,
  message,
  suggestions
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND is_opportunity = TRUE
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
ORDER BY total_elapsed_time DESC
LIMIT <LIMIT>;
```

### 5) Filter by insight topic

Focus on a category of issues. Use `<OPPORTUNITY_FILTER_CLAUSE>`: when `opportunities_only` is true use `is_opportunity = TRUE`, otherwise use `1=1`.

`insight_topic` values:

- **TABLE_SCAN** — table access (no filter, unselective filter, LIKE leading wildcard, clustering/search optimization used, etc.)
- **JOIN** — join efficiency (no join condition, inefficient condition, exploding join)
- **AGGREGATION** — inefficient aggregate
- **UNION** — unnecessary UNION DISTINCT
- **WAREHOUSE** — remote spillage, queued overload

```sql
SELECT
  query_id,
  start_time,
  total_elapsed_time / 1000 AS elapsed_seconds,
  warehouse_name,
  insight_type_id,
  is_opportunity,
  message,
  suggestions
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND insight_topic = '<INSIGHT_TOPIC>'
  AND <OPPORTUNITY_FILTER_CLAUSE>
ORDER BY total_elapsed_time DESC
LIMIT <LIMIT>;
```

### 6) Summary: most common insight types

Identify which insight types appear most often in the lookback window (useful for workload-level tuning).

```sql
SELECT
  insight_topic,
  insight_type_id,
  is_opportunity,
  COUNT(*) AS insight_count,
  COUNT(DISTINCT query_id) AS distinct_queries
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_INSIGHTS
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<WAREHOUSE_NAME_IS_NULL> OR warehouse_name = '<WAREHOUSE_NAME>')
GROUP BY insight_topic, insight_type_id, is_opportunity
ORDER BY insight_count DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) For a single query or pattern: list of insights with `insight_type_id`, `insight_topic`, `is_opportunity`, `message`, and `suggestions`
2) For time-window or topic queries: same fields, ordered by relevance (e.g. elapsed time or recency)
3) For summary: counts by `insight_topic` and `insight_type_id`
4) Brief interpretation:
   - `is_opportunity = true` = Snowflake recommends an action to improve performance
   - TABLE_SCAN insights often align with clustering/search optimization and filter selectivity
   - JOIN/AGGREGATION/UNION insights point to query rewrite opportunities
   - WAREHOUSE insights (spillage, queued overload) may warrant larger warehouse or concurrency changes

## Reference: insight types (by topic)

| insight_topic | Example insight_type_id |
|---------------|-------------------------|
| TABLE_SCAN | QUERY_INSIGHT_NO_FILTER_ON_TOP_OF_TABLE_SCAN, QUERY_INSIGHT_UNSELECTIVE_FILTER, QUERY_INSIGHT_FILTER_WITH_CLUSTERING_KEY, QUERY_INSIGHT_SEARCH_OPTIMIZATION_USED, QUERY_INSIGHT_LIKE_WITH_LEADING_WILDCARD, ... |
| JOIN | QUERY_INSIGHT_JOIN_WITH_NO_JOIN_CONDITION, QUERY_INSIGHT_INEFFICIENT_JOIN_CONDITION, QUERY_INSIGHT_EXPLODING_JOIN, ... |
| AGGREGATION (or AGGREGATE in some accounts) | QUERY_INSIGHT_INEFFICIENT_AGGREGATE |
| UNION | QUERY_INSIGHT_UNNECESSARY_UNION_DISTINCT |
| WAREHOUSE | QUERY_INSIGHT_REMOTE_SPILLAGE, QUERY_INSIGHT_QUEUED_OVERLOAD |

Full list: [Query insights types](https://docs.snowflake.com/en/user-guide/query-insights.html#label-query-insights-types).
