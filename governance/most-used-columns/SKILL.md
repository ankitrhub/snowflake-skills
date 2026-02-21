---
name: most-used-columns
description: Find the most and least queried columns using ACCESS_HISTORY to guide documentation, clustering, and deprecation.
---

# Most-Used Columns

## Goal
Quickly answer:
- Which **columns are most frequently queried** in a table or schema?
- Which **columns are never read** (candidates for deprecation)?
- Which columns are used in **filters/JOINs** (clustering key candidates)?
- How many **distinct users** access each column?

> **Requires:** Snowflake Enterprise Edition or higher (ACCESS_HISTORY is an Enterprise feature).

## Inputs to collect
- `database_name` (required)
- `schema_name` (required)
- Optional: `table_name`
- `lookback_days` (optional, default 30)
- Optional: `limit` (optional, default 50)

## Workflow

### 1) Most-queried columns (read access, ranked)

> **Note:** `ACCESS_HISTORY` has up to **3 hours** latency.
> Use `base_objects_accessed` to capture columns accessed indirectly through views.
> Column names in the JSON are stored as provided by Snowflake (typically uppercase).

```sql
SELECT
  obj.value:objectName::string    AS object_name,
  col.value:columnName::string    AS column_name,
  COUNT(DISTINCT query_id)        AS query_count,
  COUNT(DISTINCT user_name)       AS distinct_users,
  MIN(query_start_time)           AS first_access,
  MAX(query_start_time)           AS last_access
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
  LATERAL FLATTEN(input => base_objects_accessed) obj,
  LATERAL FLATTEN(input => obj.value:columns) col
WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND obj.value:objectDomain::string IN ('Table', 'View', 'MaterializedView')
  AND obj.value:objectName::string LIKE '<DATABASE_NAME>.<SCHEMA_NAME>.%'
  AND (<TABLE_NAME_IS_NULL> OR obj.value:objectName::string = '<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>')
GROUP BY object_name, column_name
ORDER BY query_count DESC
LIMIT <LIMIT>;
```

### 2) Least-queried columns (never or rarely accessed)
Compare all columns in INFORMATION_SCHEMA against ACCESS_HISTORY to find unused ones.

```sql
WITH accessed_columns AS (
  SELECT DISTINCT
    obj.value:objectName::string  AS object_name,
    col.value:columnName::string  AS column_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
    LATERAL FLATTEN(input => base_objects_accessed) obj,
    LATERAL FLATTEN(input => obj.value:columns) col
  WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    AND obj.value:objectDomain::string IN ('Table', 'View', 'MaterializedView')
    AND obj.value:objectName::string LIKE '<DATABASE_NAME>.<SCHEMA_NAME>.%'
    AND (<TABLE_NAME_IS_NULL> OR obj.value:objectName::string = '<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>')
),
all_columns AS (
  SELECT
    table_catalog || '.' || table_schema || '.' || table_name AS object_name,
    column_name
  FROM <DATABASE_NAME>.INFORMATION_SCHEMA.COLUMNS
  WHERE table_schema = '<SCHEMA_NAME>'
    AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
)
SELECT
  ac.object_name,
  ac.column_name,
  'NEVER_ACCESSED' AS status
FROM all_columns ac
LEFT JOIN accessed_columns used
  ON ac.object_name = used.object_name
 AND ac.column_name = used.column_name
WHERE used.column_name IS NULL
ORDER BY ac.object_name, ac.column_name
LIMIT <LIMIT>;
```

### 3) Column access by user (who reads what)
Useful for understanding which teams/users depend on specific columns.

```sql
SELECT
  user_name,
  obj.value:objectName::string   AS object_name,
  col.value:columnName::string   AS column_name,
  COUNT(DISTINCT query_id)       AS query_count,
  MAX(query_start_time)          AS last_access
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
  LATERAL FLATTEN(input => base_objects_accessed) obj,
  LATERAL FLATTEN(input => obj.value:columns) col
WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND obj.value:objectName::string = '<DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME>'
GROUP BY user_name, object_name, column_name
ORDER BY query_count DESC
LIMIT <LIMIT>;
```

### 4) Column usage summary per table
One row per table showing total columns, accessed columns, and coverage.

```sql
WITH accessed AS (
  SELECT DISTINCT
    obj.value:objectName::string  AS object_name,
    col.value:columnName::string  AS column_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
    LATERAL FLATTEN(input => base_objects_accessed) obj,
    LATERAL FLATTEN(input => obj.value:columns) col
  WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    AND obj.value:objectDomain::string IN ('Table')
    AND obj.value:objectName::string LIKE '<DATABASE_NAME>.<SCHEMA_NAME>.%'
),
total AS (
  SELECT
    table_catalog || '.' || table_schema || '.' || table_name AS object_name,
    COUNT(*) AS total_columns
  FROM <DATABASE_NAME>.INFORMATION_SCHEMA.COLUMNS
  WHERE table_schema = '<SCHEMA_NAME>'
  GROUP BY object_name
)
SELECT
  t.object_name,
  t.total_columns,
  COUNT(DISTINCT a.column_name) AS accessed_columns,
  t.total_columns - COUNT(DISTINCT a.column_name) AS unused_columns,
  ROUND(COUNT(DISTINCT a.column_name) / t.total_columns * 100, 1) AS coverage_pct
FROM total t
LEFT JOIN accessed a ON t.object_name = a.object_name
GROUP BY t.object_name, t.total_columns
ORDER BY unused_columns DESC;
```

## Output format
Return:
1) Most-queried columns ranked by query count and distinct users
2) Never-accessed columns (deprecation candidates)
3) Column coverage per table (% of columns actually used)
4) Recommendations:
   - Columns queried frequently in filters: consider as clustering key candidates
   - Columns never accessed in N days: candidates for deprecation or removal from models
   - Tables with low coverage %: may have schema bloat -- review with analytics team
   - High-traffic columns used by many users: prioritize documentation
