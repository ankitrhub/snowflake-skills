---
name: time-travel
description: Execute a given query with Snowflake Time Travel using AT or BEFORE with OFFSET, TIMESTAMP, or STATEMENT (query_id) to read historical data.
---

# Time Travel (Query at a Point in the Past)

Run a query as of a point in the past: "time ago" (OFFSET), exact timestamp, or relative to a specific query ID. The AT or BEFORE clause is placed in the FROM clause **immediately after** the table (or view) name.

See: [Understanding & using Time Travel](https://docs.snowflake.com/en/user-guide/data-time-travel), [AT | BEFORE](https://docs.snowflake.com/en/sql-reference/constructs/at-before).

## Goal
- Run a **given query** so results reflect **historical data** at a chosen point in time.
- Support **OFFSET** (e.g. "5 minutes ago"), **TIMESTAMP** (exact time), or **STATEMENT** (query_id).
- Produce runnable SQL with the correct AT/BEFORE clause and note retention limits.

## Inputs to collect
- **Time travel type:** one of `OFFSET`, `TIMESTAMP`, `STATEMENT` (query_id).
- **AT vs BEFORE:** only meaningful for STATEMENT; for OFFSET/TIMESTAMP use AT.
- For **OFFSET:** `offset_seconds` (e.g. 300 for 5 minutes ago); use a **negative** value (e.g. `-300` or `-60*5`).
- For **TIMESTAMP:** `timestamp_value` and optional cast (default TIMESTAMP_LTZ).
- For **STATEMENT:** `query_id` (required; query must have run within the last **14 days**).
- **Table(s) and query:** fully qualified table name(s) and the user’s SELECT/WHERE (or full query) so the clause can be applied correctly.

## Workflow

### 1) Time travel by OFFSET ("time ago")

OFFSET is the difference in **seconds** from the current time. Use a **negative** value for the past.

Template:
```sql
SELECT <COLUMNS>
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME> AT(OFFSET => -<OFFSET_SECONDS>) AS t
<WHERE_CLAUSE>;
```

Examples for the offset expression:
- 5 minutes ago: `AT(OFFSET => -60*5)`
- 1 hour ago: `AT(OFFSET => -3600)`
- 2 days ago: `AT(OFFSET => -2*24*3600)`

Full example (table as of 1 hour ago):
```sql
SELECT *
FROM my_db.my_schema.my_table AT(OFFSET => -3600) AS t
WHERE t.created_at >= '2024-01-01';
```

### 2) Time travel by TIMESTAMP (exact point in time)

The timestamp value **must be cast** to TIMESTAMP, TIMESTAMP_LTZ, TIMESTAMP_NTZ, or TIMESTAMP_TZ. Prefer **TIMESTAMP_LTZ** for session time zone.

Template:
```sql
SELECT <COLUMNS>
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME> AT(TIMESTAMP => '<TIMESTAMP_VALUE>'::TIMESTAMP_LTZ) AS t
<WHERE_CLAUSE>;
```

Example with literal timestamp:
```sql
SELECT *
FROM my_db.my_schema.my_table AT(TIMESTAMP => '2024-06-26 09:20:00 -0700'::TIMESTAMP_LTZ);
```

Example with expression ("N days ago" at run time):
```sql
SELECT *
FROM my_db.my_schema.my_table AT(TIMESTAMP => DATEADD('day', -<N_DAYS>, CURRENT_TIMESTAMP())::TIMESTAMP_LTZ);
```

### 3) Time travel by STATEMENT (query ID)

Use when the reference point is "when this statement ran." The query ID must refer to a statement executed within the **last 14 days**.

- **AT(STATEMENT => '<query_id>')** — state **including** that statement’s changes.
- **BEFORE(STATEMENT => '<query_id>')** — state **immediately before** that statement completed.

Template AT:
```sql
SELECT <COLUMNS>
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME> AT(STATEMENT => '<QUERY_ID>') AS t
<WHERE_CLAUSE>;
```

Template BEFORE:
```sql
SELECT <COLUMNS>
FROM <DATABASE_NAME>.<SCHEMA_NAME>.<TABLE_NAME> BEFORE(STATEMENT => '<QUERY_ID>') AS t
<WHERE_CLAUSE>;
```

Example:
```sql
SELECT *
FROM my_db.my_schema.my_table BEFORE(STATEMENT => '8e5d0ca9-005e-44e6-b858-a8f5b37c5726');
```

If the query is older than 14 days, use TIMESTAMP instead (see step 4).

### 4) (Optional) Resolve query_id to timestamp

When the user wants "as of when query X ran" but the query is **older than 14 days**, STATEMENT is invalid. Look up the query’s `end_time` (or `start_time`) from Account Usage and use `AT(TIMESTAMP => ...)`.

> **Note:** ACCOUNT_USAGE.QUERY_HISTORY has latency (e.g. 45–180 minutes). Use INFORMATION_SCHEMA.QUERY_HISTORY() for recent queries if needed.

```sql
SELECT
  query_id,
  start_time,
  end_time,
  total_elapsed_time,
  query_text
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_id = '<QUERY_ID>';
```

Then use the returned `end_time` in the time travel query:
```sql
SELECT *
FROM my_db.my_schema.my_table AT(TIMESTAMP => '<END_TIME>'::TIMESTAMP_LTZ);
```

(Replace `<END_TIME>` with the actual value, e.g. from a variable or literal.)

### 5) Applying time travel to a multi-table query

Each table in the FROM clause can have its own AT or BEFORE.

```sql
SELECT t1.id, t1.name, t2.amount
FROM my_db.my_schema.table_one   AT(OFFSET => -3600) AS t1
JOIN my_db.my_schema.table_two   AT(OFFSET => -3600) AS t2
  ON t1.id = t2.id;
```

If the user’s query uses a CTE that selects from a base table, the AT/BEFORE clause must be applied **inside** the CTE, not on the CTE name. Not supported: `SELECT * FROM my_cte AT(...)`. Workaround:

```sql
WITH my_cte AS (
  SELECT * FROM my_db.my_schema.my_table AT(OFFSET => -3600)
)
SELECT * FROM my_cte WHERE my_cte.flag = 'valid';
```

### 6) Retention and errors

- **Default retention:** 1 day. With Enterprise Edition (or higher), object-level retention can be set up to **90 days** via `DATA_RETENTION_TIME_IN_DAYS`.
- If the requested time is **outside the retention period** or **before the object was created**, Snowflake returns an error such as:  
  `Time travel data is not available for table <tablename>. The requested time is either beyond the allowed time travel period or before the object creation time.`
- To check retention for a table (SELECT-only path):

  ```sql
  SELECT
    table_catalog,
    table_schema,
    table_name,
    retention_time,
    created,
    deleted
  FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
  WHERE table_catalog = '<DATABASE_NAME>'
    AND table_schema = '<SCHEMA_NAME>'
    AND table_name = '<TABLE_NAME>'
  ORDER BY deleted DESC NULLS FIRST;
  ```

## Output format
Return:
1) The **final runnable SQL** with placeholders substituted, and a short note on **what point in time** it targets.
2) If using STATEMENT, whether **AT** (inclusive of that statement) or **BEFORE** (state just before completion) was used.
3) If the user provided a query_id for lookup only (e.g. older than 14 days), the **resolved timestamp** and an example `AT(TIMESTAMP => ...)` clause.
4) Remind the user of **retention** (1 day default, up to 90 days with EE) and that STATEMENT is valid only for the last 14 days.
