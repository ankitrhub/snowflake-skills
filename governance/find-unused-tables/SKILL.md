---
name: find-unused-tables
description: Find tables and views not queried in N days using ACCESS_HISTORY, and rank most/least used objects.
---

# Find Unused Tables

## Goal
Quickly answer:
- Which tables/views have **not been queried** in the last N days?
- Which tables are the **most queried** (hottest)?
- Which tables are **written to but never read**?

## Inputs to collect
- `lookback_days` (optional, default 30)
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `limit` (optional, default 100)

> **Requires:** Snowflake Enterprise Edition or higher (ACCESS_HISTORY is an Enterprise feature).
> **Latency:** ACCESS_HISTORY can be up to **3 hours** delayed.
> **Important:** Use `base_objects_accessed` (not `direct_objects_accessed`) to capture tables accessed indirectly through views.

## Workflow

### 1) Tables accessed in the last N days (base objects)
Build a set of all tables that were actually read by any query.

```sql
SELECT DISTINCT
  obj.value:objectDomain::string AS object_domain,
  obj.value:objectName::string AS object_name,
  obj.value:objectId::number AS object_id
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
  LATERAL FLATTEN(input => base_objects_accessed) obj
WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND obj.value:objectDomain::string IN ('Table', 'View', 'MaterializedView');
```

### 2) All active tables/views from ACCOUNT_USAGE.TABLES

```sql
SELECT
  table_catalog AS database_name,
  table_schema AS schema_name,
  table_name,
  table_type,
  created,
  last_ddl,
  last_ddl_by
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
  AND table_type IN ('BASE TABLE', 'VIEW', 'MATERIALIZED VIEW', 'DYNAMIC TABLE');
```

### 3) Unused tables: CTE-only LEFT JOIN to find tables with zero reads

```sql
WITH accessed_tables AS (
  SELECT DISTINCT
    obj.value:objectName::string AS object_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
    LATERAL FLATTEN(input => base_objects_accessed) obj
  WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    AND obj.value:objectDomain::string IN ('Table', 'View', 'MaterializedView')
),
all_tables AS (
  SELECT
    table_catalog AS database_name,
    table_schema AS schema_name,
    table_name,
    table_type,
    created,
    last_ddl,
    last_ddl_by
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
  WHERE deleted IS NULL
    AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
    AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
    AND table_type IN ('BASE TABLE', 'VIEW', 'MATERIALIZED VIEW', 'DYNAMIC TABLE')
)
SELECT
  t.database_name,
  t.schema_name,
  t.table_name,
  t.table_type,
  t.created,
  t.last_ddl,
  t.last_ddl_by
FROM all_tables t
LEFT JOIN accessed_tables a
  ON a.object_name = t.database_name || '.' || t.schema_name || '.' || t.table_name
WHERE a.object_name IS NULL
ORDER BY t.created
LIMIT <LIMIT>;
```

### 4) Most-queried tables (ranked by access count)
Find the hottest tables in the account.

```sql
SELECT
  obj.value:objectName::string AS object_name,
  obj.value:objectDomain::string AS object_domain,
  COUNT(DISTINCT query_id) AS query_count,
  COUNT(DISTINCT user_name) AS distinct_users,
  MIN(query_start_time) AS first_access,
  MAX(query_start_time) AS last_access
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
  LATERAL FLATTEN(input => base_objects_accessed) obj
WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND obj.value:objectDomain::string IN ('Table', 'View', 'MaterializedView')
  AND (<DATABASE_NAME_IS_NULL> OR obj.value:objectName::string LIKE '<DATABASE_NAME>.%')
GROUP BY object_name, object_domain
ORDER BY query_count DESC
LIMIT <LIMIT>;
```

### 5) (Optional) Tables written but never read
Find tables that receive writes (via `objects_modified`) but are never queried.

```sql
WITH written_tables AS (
  SELECT DISTINCT
    m.value:objectName::string AS object_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
    LATERAL FLATTEN(input => objects_modified) m
  WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    AND m.value:objectDomain::string IN ('Table')
),
read_tables AS (
  SELECT DISTINCT
    obj.value:objectName::string AS object_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
    LATERAL FLATTEN(input => base_objects_accessed) obj
  WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    AND obj.value:objectDomain::string IN ('Table')
)
SELECT w.object_name AS written_but_never_read
FROM written_tables w
LEFT JOIN read_tables r ON w.object_name = r.object_name
WHERE r.object_name IS NULL
ORDER BY w.object_name
LIMIT <LIMIT>;
```

## Output format
Return:
1) Unused tables list (database, schema, table, type, created date)
2) Most-queried tables (ranked by query count and distinct users)
3) Written-but-never-read tables (if requested)
4) Observations:
   - Tables created long ago but never queried (stale)
   - Tables only used by one user (candidate for personal schema)
5) Recommended next steps:
   - Review unused tables for potential deletion to save storage (run `get-table-sizes`)
   - Archive infrequently-accessed tables to reduce clutter
   - Check if unused views reference active tables before dropping
