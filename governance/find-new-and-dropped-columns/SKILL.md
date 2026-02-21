---
name: find-new-and-dropped-columns
description: Find dropped columns (authoritative) and new columns (schema evolution + best-effort DDL detection) using SNOWFLAKE.ACCOUNT_USAGE.
---

# Find New and Dropped Columns

## Goal
Quickly answer:
- Which **columns were dropped** recently?
- Which **new columns appeared** recently?
  - **Authoritative** for schema evolution adds (Snowpipe / COPY schema evolution)
  - **Best-effort** for DDL-based adds/renames/drops (via QUERY_HISTORY)

## Inputs to collect
- `lookback_hours` (optional, default 168)
- Optional: `database_name`
- Optional: `schema_name`
- Optional: `table_name`
- Optional: `include_system_databases` (optional, default false) — if false, filter out `SNOWFLAKE` and `SNOWFLAKE_SAMPLE_DATA`.
- Optional: `limit` (optional, default 500)

## Workflow

### 1) Dropped columns (authoritative): ACCOUNT_USAGE.COLUMNS.DELETED
`SNOWFLAKE.ACCOUNT_USAGE.COLUMNS` includes a `DELETED` timestamp when a column is dropped.

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  table_name,
  column_name,
  data_type,
  deleted AS column_deleted_at
FROM SNOWFLAKE.ACCOUNT_USAGE.COLUMNS
WHERE deleted >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
ORDER BY deleted DESC
LIMIT <LIMIT>;
```

### 2) New columns via schema evolution (authoritative when present)
Snowflake records schema evolution details in `SCHEMA_EVOLUTION_RECORD` for a table column.

This is the most reliable way (in Account Usage) to get a **time signal** for new columns.

```sql
SELECT
  table_catalog AS database_name,
  table_schema  AS schema_name,
  table_name,
  column_name,
  data_type,
  PARSE_JSON(schema_evolution_record):"EvolutionType"::string     AS evolution_type,
  PARSE_JSON(schema_evolution_record):"EvolutionMode"::string     AS evolution_mode,
  PARSE_JSON(schema_evolution_record):"FileName"::string          AS file_name,
  PARSE_JSON(schema_evolution_record):"TriggeringTime"::timestamp AS triggering_time,
  COALESCE(
    PARSE_JSON(schema_evolution_record):"QueryId"::string,
    PARSE_JSON(schema_evolution_record):"PipeId"::string
  ) AS triggering_id
FROM SNOWFLAKE.ACCOUNT_USAGE.COLUMNS
WHERE schema_evolution_record IS NOT NULL
  AND PARSE_JSON(schema_evolution_record):"EvolutionType"::string = 'ADD_COLUMN'
  AND (<INCLUDE_SYSTEM_DATABASES> OR table_catalog NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
  AND PARSE_JSON(schema_evolution_record):"TriggeringTime"::timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND (<DATABASE_NAME_IS_NULL> OR table_catalog = '<DATABASE_NAME>')
  AND (<SCHEMA_NAME_IS_NULL> OR table_schema = '<SCHEMA_NAME>')
  AND (<TABLE_NAME_IS_NULL> OR table_name = '<TABLE_NAME>')
ORDER BY triggering_time DESC
LIMIT <LIMIT>;
```

### 3) Best-effort: find column-changing DDL via QUERY_HISTORY
Account Usage does **not** provide a first-class "column created" timestamp for DDL-driven column adds.

The next best approach is:
1) find `ALTER TABLE` statements in `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`
2) optionally extract candidate column names from `QUERY_TEXT`

```sql
WITH q AS (
  SELECT
    query_id,
    user_name,
    role_name,
    database_name,
    schema_name,
    start_time,
    execution_status,
    error_message,
    query_text
  FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
  WHERE start_time >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
    AND query_text ILIKE 'ALTER TABLE%'
    AND execution_status = 'SUCCESS'
    AND (<INCLUDE_SYSTEM_DATABASES> OR database_name NOT IN ('SNOWFLAKE', 'SNOWFLAKE_SAMPLE_DATA'))
    AND (
      query_text ILIKE '% ADD COLUMN %'
      OR query_text ILIKE '% RENAME COLUMN %'
      OR query_text ILIKE '% DROP COLUMN %'
    )
    AND (<DATABASE_NAME_IS_NULL> OR database_name = '<DATABASE_NAME>')
    AND (<SCHEMA_NAME_IS_NULL> OR schema_name = '<SCHEMA_NAME>')
)
SELECT
  query_id,
  start_time,
  user_name,
  role_name,
  database_name,
  schema_name,
  execution_status,
  LEFT(query_text, 500) AS ddl_preview,
  /* Very best-effort extraction for ADD COLUMN <colname> */
  REGEXP_SUBSTR(query_text, '(?i)ADD\\s+COLUMN\\s+([A-Z0-9_"$]+)', 1, 1, 'e', 1) AS candidate_column
FROM q
ORDER BY start_time DESC
LIMIT <LIMIT>;
```

Notes:
- This does **not** guarantee correct parsing (multi-column adds, complex quoting, etc.).
- Use the returned `QUERY_ID` to investigate the full `QUERY_TEXT` and correlate with deployments.

## Output format
Return 2–3 sections:
1) Dropped columns (authoritative)
2) New columns via schema evolution (authoritative when present)
3) DDL events that likely changed columns (best-effort)
