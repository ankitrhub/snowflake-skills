---
name: monitor-pipe-failures-and-loads
description: Investigate Snowpipe failures, pipe status, and file/row load outcomes via COPY history.
---

# Monitor Pipe Failures and Loads

## Goal
Quickly answer:
- Is the pipe healthy? (paused? failing? backlog?)
- What files did it load recently?
- How many rows were loaded per file / batch?
- What errors occurred?

## Inputs to collect
- `database_name` (required)
- `schema_name` (required)
- `pipe_name` (required)
- `target_table_name` (optional but recommended; if omitted, derive from pipe definition)
- `lookback_days` (optional, default 7)
- `limit_files` (optional, default 200)

Assumption: one pipe loads into one target table.

## Workflow

### 1) Get pipe definition (and target table if needed)
Option A (SQL-only metadata; preferred in restricted MCP policies):
```sql
SELECT
  pipe_catalog,
  pipe_schema,
  pipe_name,
  definition,
  created,
  last_altered
FROM IDENTIFIER('<DATABASE_NAME>.INFORMATION_SCHEMA.PIPES')
WHERE pipe_schema = '<SCHEMA_NAME>'
  AND pipe_name = '<PIPE_NAME>';
```

Option B (if your environment allows `SHOW` commands):

```sql
SHOW PIPES LIKE '<PIPE_NAME>' IN SCHEMA <DATABASE_NAME>.<SCHEMA_NAME>;
```

Post-process with RESULT_SCAN to extract the COPY definition:

```sql
SELECT
  "name" AS pipe_name,
  "database_name" AS database_name,
  "schema_name" AS schema_name,
  "definition" AS copy_definition,
  "invalid_reason" AS invalid_reason
FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));
```

### 2) Check pipe status
`SYSTEM$PIPE_STATUS` returns JSON with state/backlog/error details.

```sql
SELECT SYSTEM$PIPE_STATUS('<DATABASE_NAME>.<SCHEMA_NAME>.<PIPE_NAME>') AS pipe_status;
```

### 3) Files loaded + row counts + errors (COPY history)

If you know the target table:

> **Note:** The `TABLE_NAME` parameter is a simple (unqualified) table name, not a fully-qualified name.
> The session or function path must be scoped to the correct database.

```sql
SELECT *
FROM TABLE(<DATABASE_NAME>.INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => '<TARGET_TABLE_NAME>',
  START_TIME => DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
))
ORDER BY LAST_LOAD_TIME DESC
LIMIT <LIMIT_FILES>;
```

Fallback (Account Usage; more history, can be delayed):
```sql
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  AND table_catalog = '<DATABASE_NAME>'
  AND table_schema = '<SCHEMA_NAME>'
  AND table_name = '<TARGET_TABLE_NAME>'
ORDER BY last_load_time DESC
LIMIT <LIMIT_FILES>;
```

### 4) (Optional) Pipe usage trend

> **Note:** `PIPE_USAGE_HISTORY` is an account-level Information Schema function.
> Use the `PIPE_NAME` argument (fully-qualified) to filter.
> Output columns: `START_TIME`, `END_TIME`, `PIPE_NAME`, `CREDITS_USED`, `BYTES_INSERTED`, `FILES_INSERTED`.

```sql
SELECT *
FROM TABLE(SNOWFLAKE.INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
  DATE_RANGE_START => DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_DATE()),
  DATE_RANGE_END => CURRENT_DATE(),
  PIPE_NAME => '<DATABASE_NAME>.<SCHEMA_NAME>.<PIPE_NAME>'
))
ORDER BY START_TIME DESC;
```

## Output format
Return:
1) Pipe status summary (state/backlog/errors from SYSTEM$PIPE_STATUS)
2) Recent COPY history rows (file name, rows loaded, errors)
3) Top failing files grouped by error message
