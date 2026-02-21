---
name: find-recent-deletions
description: Find recently dropped tables/views and columns using SNOWFLAKE.ACCOUNT_USAGE.
---

# Find Recent Deletions (Dropped Objects & Columns)

## Goal
Quickly answer:
- Which tables/views were **dropped recently**?
- Which columns were **dropped recently**?

## Inputs to collect
- `lookback_hours` (optional, default 168)
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `include_system_databases` (optional, default false) — if false, filter out `SNOWFLAKE` and `SNOWFLAKE_SAMPLE_DATA`.
- Optional: `limit` (optional, default 500)

## Workflow

### 1) Dropped tables/views: ACCOUNT_USAGE.TABLES.DELETED
`SNOWFLAKE.ACCOUNT_USAGE.TABLES` includes a `DELETED` timestamp when the object is dropped.

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  table_name,
  table_type,
  deleted  AS dropped_at,
  last_ddl AS last_ddl_at,
  last_ddl_by
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
ORDER BY deleted DESC
LIMIT <LIMIT>;
```

### 2) Dropped views specifically (optional)
If you prefer a view-only report (and want view-specific metadata like `VIEW_DEFINITION`), you can query:

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  table_name    AS view_name,
  deleted       AS dropped_at,
  last_ddl      AS last_ddl_at,
  last_ddl_by
FROM SNOWFLAKE.ACCOUNT_USAGE.VIEWS
WHERE deleted >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
ORDER BY deleted DESC
LIMIT <LIMIT>;
```

### 3) Dropped columns: ACCOUNT_USAGE.COLUMNS.DELETED

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  table_name,
  column_name,
  data_type,
  deleted AS column_dropped_at
FROM SNOWFLAKE.ACCOUNT_USAGE.COLUMNS
WHERE deleted >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
ORDER BY deleted DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) Dropped tables/views (with dropped_at, last_ddl_by)
2) Dropped columns

## Caveats
- Account Usage views have latency (commonly **45–180 minutes**, varies by view).
- If objects are dropped and recreated with the same name, use the internal IDs (`TABLE_ID`, `COLUMN_ID`) when you need to disambiguate.
