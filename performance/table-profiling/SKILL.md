---
name: table-profiling
description: Lightweight data profiling for a table -- row count, column stats, null rates, cardinality, and sample rows using sampling to limit cost.
---

# Table Profiling (Lightweight)

## Goal
Quickly answer:
- How many **rows** does this table have?
- What are the **columns, data types, and nullability**?
- What is the **null rate and approximate cardinality** per column?
- What does a **sample of the data** look like?

> **Important:** This skill runs actual queries against the table data.
> All profiling queries use **sampling or limits** to keep cost low.
> Do NOT run full-table scans -- always use `TABLESAMPLE` or `LIMIT`.

## Inputs to collect
- `database_name` (required)
- `schema_name` (required)
- `table_name` (required)
- `sample_pct` (optional, default 1) -- percentage of table to sample for stats (1-10 recommended)
- `sample_rows` (optional, default 20) -- number of sample rows to return for preview

## Workflow

### 1) Table metadata: row count, size, creation date
Use Information Schema first (SELECT-only, no table scan).

```sql
SELECT
  table_catalog AS database_name,
  table_schema AS schema_name,
  table_name,
  table_type AS table_kind,
  row_count AS approx_row_count,
  bytes,
  ROUND(bytes / POWER(1024, 3), 3) AS size_gb,
  created AS created_on,
  clustering_key AS cluster_by,
  comment AS table_comment
FROM <DATABASE_NAME>.INFORMATION_SCHEMA.TABLES
WHERE table_schema = '<SCHEMA_NAME>'
  AND table_name = '<TABLE_NAME>';
```

Optional fallback (if your environment allows `SHOW` commands):

```sql
SHOW TABLES LIKE '<TABLE_NAME>' IN SCHEMA <DATABASE_NAME>.<SCHEMA_NAME>;
```

```sql
SELECT
  "database_name" AS database_name,
  "schema_name" AS schema_name,
  "name" AS table_name,
  "kind" AS table_kind,
  "rows" AS approx_row_count,
  "bytes" AS bytes,
  ROUND("bytes" / POWER(1024, 3), 3) AS size_gb,
  "created_on" AS created_on,
  "cluster_by" AS cluster_by,
  "comment" AS table_comment
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

> **Note:** Row counts from SHOW TABLES are estimates maintained by Snowflake. They are usually accurate
> but may lag slightly after large DML operations.

### 2) Column listing: names, types, nullability, defaults
Uses INFORMATION_SCHEMA (no table scan).

```sql
SELECT
  ordinal_position,
  column_name,
  data_type,
  is_nullable,
  column_default,
  character_maximum_length,
  numeric_precision,
  numeric_scale,
  comment
FROM <DATABASE_NAME>.INFORMATION_SCHEMA.COLUMNS
WHERE table_schema = '<SCHEMA_NAME>'
  AND table_name = '<TABLE_NAME>'
ORDER BY ordinal_position;
```

### 3) Column-level stats: null rate + approximate cardinality (SAMPLED)
Uses `TABLESAMPLE` to profile a small percentage of the table.

> **Cost control:** `TABLESAMPLE SYSTEM (<sample_pct>)` scans only a fraction of micro-partitions.
> For large tables, 1% is usually sufficient. Increase to 5-10% for small tables.

Generate and run this query dynamically for all columns (or a subset):

```sql
SELECT
  COUNT(*) AS sampled_rows,
  -- Repeat the following block for each column to profile:
  -- Column: <COLUMN_NAME>
  ROUND(SUM(CASE WHEN <COLUMN_NAME> IS NULL THEN 1 ELSE 0 END)::FLOAT / COUNT(*) * 100, 2)
    AS <COLUMN_NAME>_null_pct,
  APPROX_COUNT_DISTINCT(<COLUMN_NAME>) AS <COLUMN_NAME>_approx_distinct,
  MIN(<COLUMN_NAME>)::VARCHAR AS <COLUMN_NAME>_min,
  MAX(<COLUMN_NAME>)::VARCHAR AS <COLUMN_NAME>_max
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>
  TABLESAMPLE SYSTEM (<SAMPLE_PCT>);
```

For tables with many columns, **limit profiling to the first 20 columns** (by ordinal position)
or to a user-specified subset to avoid very wide result sets.

### 4) Sample rows (preview)
Return a small number of rows for visual inspection.

```sql
SELECT *
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>
TABLESAMPLE BERNOULLI (<SAMPLE_PCT>)
LIMIT <SAMPLE_ROWS>;
```

> If `TABLESAMPLE` returns fewer rows than requested (very small sample_pct on a small table),
> fall back to a simple `LIMIT`:
> ```sql
> SELECT * FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME> LIMIT <SAMPLE_ROWS>;
> ```

### 5) (Optional) Value frequency for a specific column
If a user asks about the distribution of a specific column, use a top-N frequency count.

```sql
SELECT
  <COLUMN_NAME>,
  COUNT(*) AS frequency,
  ROUND(COUNT(*)::FLOAT / SUM(COUNT(*)) OVER () * 100, 2) AS pct
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>
  TABLESAMPLE SYSTEM (<SAMPLE_PCT>)
GROUP BY <COLUMN_NAME>
ORDER BY frequency DESC
LIMIT 20;
```

## Cost guardrails
- **Always** use `TABLESAMPLE` or `LIMIT` -- never scan the full table.
- Default `sample_pct` is **1%**. For tables under 1M rows, increase to 10%.
- Profile at most **20 columns** by default. Let the user request more if needed.
- For very large tables (>1TB), use `TABLESAMPLE SYSTEM (0.1)` (0.1%).
- Cast MIN/MAX to VARCHAR to avoid type errors on semi-structured columns.

## Output format
Return:
1) Table metadata (row count, size, creation date, clustering key)
2) Column listing (name, type, nullable)
3) Per-column stats (null %, approx distinct count, min, max) -- from sampled data
4) Sample rows (for visual inspection)
5) Observations:
   - Columns with high null rates (>50%): may indicate optional or poorly populated fields
   - Columns with cardinality = 1: may be constant / useless for filtering
   - Columns with cardinality close to row count: likely unique identifiers
   - Note that all stats are approximate (based on <sample_pct>% sample)
