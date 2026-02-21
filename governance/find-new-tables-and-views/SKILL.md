---
name: find-new-tables-and-views
description: Find newly created tables/views (and optionally recently altered ones) using SNOWFLAKE.ACCOUNT_USAGE.
---

# Find New Tables and Views

## Goal
Quickly answer:
- What **new tables/views** were created recently?
- (Optional) What objects had **recent DDL changes**?
- Who executed the last DDL (best-effort via `LAST_DDL_BY`)?

## Inputs to collect
- `lookback_hours` (optional, default 168)
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `name_like` (SQL ILIKE pattern, e.g. `FACT_%`)
- Optional: `include_table_types` (optional; default: BASE TABLE, VIEW, MATERIALIZED VIEW, DYNAMIC TABLE)
- Optional: `include_deleted` (optional, default false)
- Optional: `include_system_databases` (optional, default false) — if false, filter out `SNOWFLAKE` and `SNOWFLAKE_SAMPLE_DATA`.
- Optional: `limit` (optional, default 500)

## Workflow

### 1) Newly created tables/views (ACCOUNT_USAGE.TABLES)
`SNOWFLAKE.ACCOUNT_USAGE.TABLES` includes both tables and views (via `TABLE_TYPE`).

Notes:
- Account Usage views can have **latency (often up to ~90 minutes)**.
- Filter out dropped objects with `deleted IS NULL` unless you explicitly want deletions.

```sql
WITH objects AS (
  SELECT
    table_catalog AS database_name,
    table_schema  AS schema_name,
    table_name,
    table_type,
    created,
    last_ddl,
    last_ddl_by,
    deleted
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
  WHERE created >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
    AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
    AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
    AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
    AND (<NAME_LIKE_IS_NULL> OR table_name ILIKE '<NAME_LIKE>')
    AND (
      <INCLUDE_TABLE_TYPES_IS_NULL>
      OR table_type IN (<INCLUDE_TABLE_TYPES_CSV>)
    )
    AND (<INCLUDE_DELETED> OR deleted IS NULL)
)
SELECT *
FROM objects
ORDER BY created DESC
LIMIT <LIMIT>;
```

### 2) (Optional) Include view definitions for plain views
If you need the SQL definition for newly created views, join to `SNOWFLAKE.ACCOUNT_USAGE.VIEWS`.

```sql
WITH new_objects AS (
  SELECT
    table_catalog AS database_name,
    table_schema  AS schema_name,
    table_name,
    table_type,
    created,
    last_ddl,
    last_ddl_by,
    deleted
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
  WHERE created >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
    AND table_type = 'VIEW'
    AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
    AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
    AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
    AND (<NAME_LIKE_IS_NULL> OR table_name ILIKE '<NAME_LIKE>')
    AND (<INCLUDE_DELETED> OR deleted IS NULL)
)
SELECT
  o.database_name,
  o.schema_name,
  o.table_name AS view_name,
  o.created,
  o.last_ddl,
  o.last_ddl_by,
  LEFT(v.view_definition, 2000) AS view_definition_preview
FROM new_objects o
JOIN SNOWFLAKE.ACCOUNT_USAGE.VIEWS v
  ON v.table_catalog = o.database_name
 AND v.table_schema  = o.schema_name
 AND v.table_name    = o.table_name
ORDER BY o.created DESC
LIMIT <LIMIT>;
```

### 3) (Optional) Recently altered objects (DDL activity)
If you also want “recent DDL changes” (not just creates), use `LAST_DDL` instead of `CREATED`:

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  table_name,
  table_type,
  created,
  last_ddl,
  last_ddl_by,
  deleted
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE last_ddl >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
  AND (<NAME_LIKE_IS_NULL> OR table_name ILIKE '<NAME_LIKE>')
  AND (<INCLUDE_DELETED> OR deleted IS NULL)
ORDER BY last_ddl DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) Newly created objects table: (database, schema, name, type, created, last_ddl_by)
2) Optional: view definition previews for new views
3) Optional: recently altered objects table
