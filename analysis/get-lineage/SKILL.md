---
name: get-lineage
description: Retrieve upstream or downstream data lineage for a Snowflake object (column, table, stage, dataset, or module) using GET_LINEAGE, with an optional ACCESS_HISTORY deep dive.
---

# Get Lineage (Snowflake)

## Inputs to collect
- `object_name` (required) — fully qualified recommended (e.g. `DB.SCHEMA.OBJECT` or `DB.SCHEMA.TABLE.COLUMN`)
- `object_domain` (required) — one of: `COLUMN`, `TABLE`, `STAGE`, `DATASET`, `MODULE`
- `direction` (required) — `UPSTREAM` or `DOWNSTREAM`
- `distance` (optional, default **5**, max **5**)

Optional filters for the deep dive:
- `lookback_days` (default 30)
- `query_id` or `root_query_id` (if you want lineage evidence for a specific pipeline run)
- `user_name`, `role_name`, `warehouse_name`, `query_tag`

## Workflow

## 1) Primary: `SNOWFLAKE.CORE.GET_LINEAGE`

> **Requires:** Snowflake Enterprise Edition or higher.
> **Limits:** Returns at most **10 million rows** (silently truncated if exceeded).

`GET_LINEAGE` returns an **edge list** (SOURCE → TARGET) with a `DISTANCE` column and a `PROCESS` VARIANT that can contain metadata such as a query id.

### 1.1) Basic lineage query

```sql
SELECT *
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE(
    '<OBJECT_NAME>',
    '<OBJECT_DOMAIN>',
    '<DIRECTION>',
    <DISTANCE>
  )
);
```

Where:
- `<OBJECT_DOMAIN>` is `COLUMN`, `TABLE` (includes table-like objects, views, dynamic tables), `STAGE`, `DATASET`, or `MODULE` (ML models).
- `<DIRECTION>` is `UPSTREAM` or `DOWNSTREAM`.
- `<DISTANCE>` defaults to **5**; maximum is **5**.

### 1.2) Most common use-cases

#### Column lineage (UPSTREAM)
```sql
SELECT *
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE(
    'MY_DB.MY_SCHEMA.MY_TABLE.MY_COLUMN',
    'COLUMN',
    'UPSTREAM',
    3
  )
);
```

#### Column lineage (DOWNSTREAM)
```sql
SELECT *
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE(
    'MY_DB.MY_SCHEMA.MY_TABLE.MY_COLUMN',
    'COLUMN',
    'DOWNSTREAM',
    3
  )
);
```

#### Table lineage (UPSTREAM)
```sql
SELECT *
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE(
    'MY_DB.MY_SCHEMA.MY_TABLE',
    'TABLE',
    'UPSTREAM',
    3
  )
);
```

#### Table lineage (DOWNSTREAM)
```sql
SELECT *
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE(
    'MY_DB.MY_SCHEMA.MY_TABLE',
    'TABLE',
    'DOWNSTREAM',
    3
  )
);
```

### 1.3) Optional: filter to only column-to-column edges
```sql
SELECT *
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE('<OBJECT_NAME>', '<OBJECT_DOMAIN>', '<DIRECTION>', <DISTANCE>)
)
WHERE SOURCE_OBJECT_DOMAIN = 'COLUMN'
  AND TARGET_OBJECT_DOMAIN = 'COLUMN';
```

### 1.4) Optional: edge list suitable for graphing
```sql
SELECT
  SOURCE_OBJECT_DATABASE || '.' || SOURCE_OBJECT_SCHEMA || '.' || SOURCE_OBJECT_NAME
    || IFF(SOURCE_OBJECT_DOMAIN = 'COLUMN', '.' || SOURCE_COLUMN_NAME, '') AS source_node,
  TARGET_OBJECT_DATABASE || '.' || TARGET_OBJECT_SCHEMA || '.' || TARGET_OBJECT_NAME
    || IFF(TARGET_OBJECT_DOMAIN = 'COLUMN', '.' || TARGET_COLUMN_NAME, '') AS target_node,
  DISTANCE,
  PROCESS
FROM TABLE(
  SNOWFLAKE.CORE.GET_LINEAGE('<OBJECT_NAME>', '<OBJECT_DOMAIN>', '<DIRECTION>', <DISTANCE>)
);
```

## 2) Deep dive: `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` (column-level evidence)

Use this when you want to prove lineage based on *actual write activity*.

Key idea: `ACCESS_HISTORY.objects_modified` contains a `columns` array; each target column can include:
- `directSources`
- `baseSources`

These arrays can be flattened to build a **target_column ← source_column** edge list.

> Notes / constraints
> - Enterprise Edition feature; latency can be up to ~180 minutes.
> - Not every query shape is logged; failed queries are not logged.

### 2.1) Build column lineage edges for writes into a specific table

Replace `<TARGET_DB>.<TARGET_SCHEMA>.<TARGET_TABLE>` and adjust lookback as needed.

```sql
WITH ah AS (
  SELECT
    query_id,
    query_start_time,
    user_name,
    direct_objects_accessed,
    base_objects_accessed,
    objects_modified
  FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
  WHERE query_start_time >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
),
mods AS (
  SELECT
    ah.query_id,
    ah.query_start_time,
    ah.user_name,
    m.value:objectName::string AS target_object_name,
    m.value:objectDomain::string AS target_object_domain,
    c.value:columnName::string AS target_column_name,
    bs.value:objectName::string AS source_object_name,
    bs.value:objectDomain::string AS source_object_domain,
    bs.value:columnName::string AS source_column_name,
    'baseSources' AS source_kind
  FROM ah,
    LATERAL FLATTEN(input => ah.objects_modified) m,
    LATERAL FLATTEN(input => m.value:columns) c,
    LATERAL FLATTEN(input => c.value:baseSources) bs
  WHERE target_object_name = '<TARGET_DB>.<TARGET_SCHEMA>.<TARGET_TABLE>'

  UNION ALL

  SELECT
    ah.query_id,
    ah.query_start_time,
    ah.user_name,
    m.value:objectName::string AS target_object_name,
    m.value:objectDomain::string AS target_object_domain,
    c.value:columnName::string AS target_column_name,
    ds.value:objectName::string AS source_object_name,
    ds.value:objectDomain::string AS source_object_domain,
    ds.value:columnName::string AS source_column_name,
    'directSources' AS source_kind
  FROM ah,
    LATERAL FLATTEN(input => ah.objects_modified) m,
    LATERAL FLATTEN(input => m.value:columns) c,
    LATERAL FLATTEN(input => c.value:directSources) ds
  WHERE target_object_name = '<TARGET_DB>.<TARGET_SCHEMA>.<TARGET_TABLE>'
)
SELECT
  query_id,
  query_start_time,
  user_name,
  target_object_name,
  target_column_name,
  source_object_name,
  source_column_name,
  source_kind
FROM mods
ORDER BY query_start_time DESC;
```

### 2.2) If you have a query_id/root_query_id
Filter `ACCESS_HISTORY` by `query_id` (or `root_query_id` if you're correlating a chain). Then run the same flattening.

## Output format
Return:
1) A small summary: object, direction, distance, and whether lineage was found
2) Top edges (source → target) grouped by `DISTANCE`
3) A graph-friendly edge list (source_node, target_node)
4) If needed, the ACCESS_HISTORY "evidence" table for the most recent writes
