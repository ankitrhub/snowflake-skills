---
name: stream-has-data-and-staleness
description: Check whether a Snowflake stream has data pending and whether it is stale.
---

# Stream Has Data and Staleness

## Goal
Quickly answer:
- Does a stream currently have unconsumed change data?
- Is the stream stale (cannot be queried) or approaching staleness?

## Inputs to collect
- `database_name` (required)
- `schema_name` (required)
- `stream_name` (optional; if omitted, evaluate all streams in the schema)

## Workflow

### 1) List streams in schema (SELECT-only metadata path)
```sql
SELECT
  stream_name,
  stream_catalog AS database_name,
  stream_schema AS schema_name,
  table_name AS source_table_name,
  stale AS is_stale,
  stale_after,
  DATEDIFF('hour', CURRENT_TIMESTAMP(), stale_after) AS hours_until_stale
FROM <DATABASE_NAME>.INFORMATION_SCHEMA.STREAMS
WHERE stream_schema = '<SCHEMA_NAME>'
  AND (<STREAM_NAME_IS_NULL> OR stream_name = '<STREAM_NAME>')
ORDER BY stale DESC, stale_after;
```

Optional fallback (if your environment allows `SHOW` commands):

```sql
SHOW STREAMS IN SCHEMA <DATABASE_NAME>.<SCHEMA_NAME>;
```

### 2) Check if a specific stream has data
`SYSTEM$STREAM_HAS_DATA` returns TRUE/FALSE.

```sql
SELECT SYSTEM$STREAM_HAS_DATA('<DATABASE_NAME>.<SCHEMA_NAME>.<STREAM_NAME>') AS has_data;
```

### 3) (Optional) Find stale streams quickly
Run the metadata query above and filter `is_stale = TRUE`.

## Output format
Return:
1) Stream metadata (stale, stale_after)
2) `has_data` (if stream_name provided)
3) Next steps:
   - If stale: recreate stream (cannot be consumed until recreated)
   - If has_data: check consumer task health (run `monitor-task-failures`)
