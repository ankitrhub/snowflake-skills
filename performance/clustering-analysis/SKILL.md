---
name: clustering-analysis
description: Analyze clustering key effectiveness, reclustering costs, and partition pruning efficiency using SYSTEM$CLUSTERING_INFORMATION, AUTOMATIC_CLUSTERING_HISTORY, TABLE_PRUNING_HISTORY, and TABLE_QUERY_PRUNING_HISTORY.
---

# Clustering, Pruning & Reclustering Analysis

## Goal
Quickly answer:
- Is my **clustering key effective** (low average depth, high constant partition ratio)?
- How much is **automatic reclustering costing** in credits?
- Which tables consume the **most reclustering credits**?
- Is a specific table being **reclustered repeatedly** (over-churning)?
- Should I **add, change, or drop** a clustering key?
- Which tables have the **worst pruning efficiency** (scanning too many partitions)?
- Which **query patterns** are not benefiting from clustering (low pruning ratio per query hash)?
- Is pruning improving or degrading **over time** for a table?

## Inputs to collect
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `table_name` (required for clustering info check)
- Optional: `column_expressions` (for evaluating alternative clustering keys)
- `lookback_days` (optional, default 14)
- Optional: `limit` (optional, default 20)

## Workflow

### 1) Check clustering quality for a specific table

> **Note:** `SYSTEM$CLUSTERING_INFORMATION` returns JSON.
> Arguments must be enclosed in **single quotes**, including column expressions.
> If a table has no clustering key, this function can return an error (for example `000005 ... table is not clustered`).

Pre-check whether the table currently has a clustering key:

```sql
SELECT
  table_catalog AS database_name,
  table_schema AS schema_name,
  table_name,
  clustering_key
FROM <DATABASE_NAME>.INFORMATION_SCHEMA.TABLES
WHERE table_schema = '<SCHEMA_NAME>'
  AND table_name = '<TABLE_NAME>';
```

If `clustering_key` is NULL, skip `SYSTEM$CLUSTERING_INFORMATION` and use sections 6-8 (pruning analysis) to evaluate whether clustering should be added.

#### With the table's defined clustering key:
```sql
SELECT PARSE_JSON(SYSTEM$CLUSTERING_INFORMATION('<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>')) AS clustering_info;
```

#### With specific columns (to evaluate an alternative key):
```sql
SELECT PARSE_JSON(SYSTEM$CLUSTERING_INFORMATION('<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>', '(<COLUMN_EXPRESSIONS>)')) AS clustering_info;
```

Post-process the JSON to extract key metrics:
```sql
WITH info AS (
  SELECT PARSE_JSON(SYSTEM$CLUSTERING_INFORMATION('<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>')) AS ci
)
SELECT
  ci:cluster_by_keys::string             AS cluster_by_keys,
  ci:total_partition_count::number       AS total_partitions,
  ci:total_constant_partition_count::number AS constant_partitions,
  ROUND(ci:total_constant_partition_count / NULLIF(ci:total_partition_count, 0) * 100, 1)
                                          AS constant_partition_pct,
  ci:average_overlaps::number            AS avg_overlaps,
  ci:average_depth::number               AS avg_depth,
  ci:notes::string                       AS notes
FROM info;
```

**Interpretation:**
- `avg_depth` close to **1** = well-clustered (ideal for pruning)
- `avg_depth` > **5** = poorly clustered; filters on these columns scan many partitions
- `constant_partition_pct` close to **100%** = excellent; micro-partitions contain non-overlapping ranges
- `avg_overlaps` close to **0** = minimal overlap between partitions
- `notes` may contain Snowflake's own suggestions

### 2) Top tables by reclustering credit cost

> **Note:** `AUTOMATIC_CLUSTERING_HISTORY` has up to **3 hours** latency.
> Rows may be reclustered multiple times, so `NUM_ROWS_RECLUSTERED` can exceed the total table row count.

```sql
SELECT
  database_name,
  schema_name,
  table_name,
  SUM(credits_used) AS total_credits,
  SUM(num_bytes_reclustered) / POWER(1024, 3) AS total_gb_reclustered,
  SUM(num_rows_reclustered) AS total_rows_reclustered,
  COUNT(*) AS reclustering_events
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY database_name, schema_name, table_name
ORDER BY total_credits DESC
LIMIT <LIMIT>;
```

### 3) Daily reclustering trend for a table
Track whether reclustering is increasing over time (which may indicate growing DML volume).

```sql
SELECT
  DATE_TRUNC('day', start_time) AS clustering_date,
  database_name,
  schema_name,
  table_name,
  SUM(credits_used) AS daily_credits,
  SUM(num_bytes_reclustered) / POWER(1024, 3) AS daily_gb_reclustered,
  SUM(num_rows_reclustered) AS daily_rows_reclustered,
  COUNT(*) AS daily_events
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY clustering_date, database_name, schema_name, table_name
ORDER BY clustering_date DESC;
```

### 4) Correlate reclustering with DML activity
Join with `TABLE_DML_HISTORY` to see if reclustering spikes follow DML spikes.

```sql
SELECT
  DATE_TRUNC('day', c.start_time) AS activity_date,
  c.table_name,
  SUM(c.credits_used) AS clustering_credits,
  SUM(c.num_rows_reclustered) AS rows_reclustered,
  SUM(d.rows_added + d.rows_removed + d.rows_updated) AS dml_rows
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY c
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TABLE_DML_HISTORY d
  ON c.table_id = d.table_id
 AND DATE_TRUNC('day', c.start_time) = DATE_TRUNC('day', d.start_time)
WHERE c.start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND c.database_name = '<DATABASE_NAME>'
  AND c.schema_name = '<SCHEMA_NAME>'
  AND c.table_name = '<TABLE_NAME>'
GROUP BY activity_date, c.table_name
ORDER BY activity_date DESC;
```

### 5) (Optional) Compare clustering quality of current key vs alternative
Run `SYSTEM$CLUSTERING_INFORMATION` with different column sets and compare `avg_depth`.

```sql
SELECT
  'current_key' AS key_type,
  ci:cluster_by_keys::string AS columns,
  ci:average_depth::number AS avg_depth,
  ci:average_overlaps::number AS avg_overlaps,
  ci:total_constant_partition_count::number / NULLIF(ci:total_partition_count::number, 0) * 100 AS constant_pct
FROM (SELECT PARSE_JSON(SYSTEM$CLUSTERING_INFORMATION('<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>')) AS ci)

UNION ALL

SELECT
  'alternative_key' AS key_type,
  ci:cluster_by_keys::string AS columns,
  ci:average_depth::number AS avg_depth,
  ci:average_overlaps::number AS avg_overlaps,
  ci:total_constant_partition_count::number / NULLIF(ci:total_partition_count::number, 0) * 100 AS constant_pct
FROM (SELECT PARSE_JSON(SYSTEM$CLUSTERING_INFORMATION('<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>', '(<ALT_COLUMN_EXPRESSIONS>)')) AS ci);
```

### 6) Worst-pruned tables (TABLE_PRUNING_HISTORY)

Identify tables where queries scan many partitions and prune few — candidates for clustering key or search optimization.

> **Note:** `TABLE_PRUNING_HISTORY` has up to **6 hours** latency. This view does not include pruning information for hybrid tables.

```sql
SELECT
  table_id,
  ANY_VALUE(table_name) AS table_name,
  ANY_VALUE(schema_name) AS schema_name,
  ANY_VALUE(database_name) AS database_name,
  SUM(num_scans) AS total_num_scans,
  SUM(partitions_scanned) AS total_partitions_scanned,
  SUM(partitions_pruned) AS total_partitions_pruned,
  SUM(rows_scanned) AS total_rows_scanned,
  SUM(rows_pruned) AS total_rows_pruned,
  SUM(partitions_pruned) / GREATEST(SUM(partitions_scanned) + SUM(partitions_pruned), 1) AS pruning_ratio
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_PRUNING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
GROUP BY table_id
ORDER BY pruning_ratio ASC, total_partitions_scanned DESC
LIMIT <LIMIT>;
```

**Interpretation:** Lower `pruning_ratio` = worse (queries scan more partitions). Order by ratio ascending to see worst first.

### 7) Query-level pruning analysis (TABLE_QUERY_PRUNING_HISTORY)

Identify specific query patterns (by query hash and warehouse) that have poor pruning on a table — high-impact optimization targets.

> **Note:** `TABLE_QUERY_PRUNING_HISTORY` has up to **4 hours** latency. Does not include hybrid tables.

```sql
SELECT
  database_name,
  schema_name,
  table_name,
  warehouse_name,
  query_hash,
  query_parameterized_hash,
  SUM(num_queries) AS total_queries,
  SUM(aggregate_query_execution_time) AS sum_execution_time_ms,
  SUM(partitions_scanned) AS total_partitions_scanned,
  SUM(partitions_pruned) AS total_partitions_pruned,
  SUM(partitions_pruned) / GREATEST(SUM(partitions_scanned) + SUM(partitions_pruned), 1) AS pruning_ratio,
  SUM(partitions_scanned) / NULLIF(SUM(num_queries), 0) AS partitions_scanned_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_QUERY_PRUNING_HISTORY
WHERE interval_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY database_name, schema_name, table_name, warehouse_name, query_hash, query_parameterized_hash
ORDER BY pruning_ratio ASC, sum_execution_time_ms DESC
LIMIT <LIMIT>;
```

**Interpretation:** Rows with low `pruning_ratio` and high `sum_execution_time_ms` are the best candidates to optimize (e.g., add clustering key or change filter columns).

### 8) Pruning trend over time

Track daily pruning ratio and scan volume for a table — useful for before/after analysis when enabling clustering or search optimization.

```sql
SELECT
  DATE_TRUNC('day', start_time) AS pruning_date,
  table_id,
  ANY_VALUE(table_name) AS table_name,
  ANY_VALUE(schema_name) AS schema_name,
  ANY_VALUE(database_name) AS database_name,
  SUM(num_scans) AS total_num_scans,
  SUM(partitions_scanned) AS total_partitions_scanned,
  SUM(partitions_pruned) AS total_partitions_pruned,
  SUM(partitions_pruned) / GREATEST(SUM(partitions_scanned) + SUM(partitions_pruned), 1) AS pruning_ratio
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_PRUNING_HISTORY
WHERE start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
GROUP BY pruning_date, table_id
ORDER BY pruning_date DESC;
```

**Interpretation:** Improving pruning ratio over time after enabling clustering validates that the key is correct.

## Output format
Return:
1) Clustering quality metrics for the requested table (avg_depth, overlaps, constant %)
2) Top tables by reclustering credit cost
3) Daily reclustering trend
4) DML correlation (if applicable)
5) Compare clustering quality of current key vs alternative (if requested)
6) Worst-pruned tables (pruning ratio, partitions/rows scanned vs pruned)
7) Query-level pruning analysis (query hash, warehouse, pruning ratio, execution time)
8) Pruning trend over time (if table specified)
9) Recommendations:
   - **Clustering:** `avg_depth` > 5 with high query frequency on those columns: add or change clustering key
   - **Clustering:** `avg_depth` close to 1: clustering is effective; no changes needed
   - **Clustering:** High reclustering credits but low DML: may indicate an overly aggressive or misaligned key
   - **Clustering:** High DML + high reclustering: consider batching writes or adjusting clustering strategy
   - **Clustering:** If alternative key has significantly lower avg_depth: consider ALTER TABLE ... CLUSTER BY
   - **Clustering:** Tables with clustering credits but rarely queried: consider dropping the clustering key to save cost
   - **Pruning:** `pruning_ratio` < 0.5 = poor; queries scan most partitions — candidate for clustering key or key change
   - **Pruning:** `pruning_ratio` > 0.9 = excellent; clustering is paying off
   - **Pruning:** Specific query hashes with low pruning but high execution time = high-impact optimization targets
   - **Pruning:** Improving pruning trend after enabling clustering = validation that the key is correct
