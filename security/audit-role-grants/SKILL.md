---
name: audit-role-grants
description: Audit role assignments and privilege grants to detect over-privileged users, orphaned grants, and recent changes.
---

# Audit Role Grants

## Goal
Quickly answer:
- Which users hold **high-privilege roles** (ACCOUNTADMIN, SYSADMIN, SECURITYADMIN)?
- Which roles have **broad privileges** (OWNERSHIP, ALL PRIVILEGES) on many objects?
- What role grants were **recently added or revoked**?
- Are there **separation-of-duties** violations (same user with conflicting roles)?

## Inputs to collect
- Optional: `user_name`
- Optional: `role_name`
- Optional: `lookback_days` (default 30, for recent changes)
- Optional: `limit` (optional, default 100)

## Workflow

### 1) Users with high-privilege roles

> **Note:** `GRANTS_TO_USERS` has up to **2 hours** latency.
> It tracks role-to-user assignments (including historical revocations via `DELETED_ON`).
> Active grants have `DELETED_ON IS NULL`.

```sql
SELECT
  grantee_name AS user_name,
  role,
  granted_by,
  created_on AS granted_at
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE deleted_on IS NULL
  AND role IN ('ACCOUNTADMIN', 'SYSADMIN', 'SECURITYADMIN', 'USERADMIN')
  AND (<USER_NAME_IS_NULL> OR grantee_name = '<USER_NAME>')
ORDER BY role, grantee_name;
```

### 2) All active role assignments for a user

```sql
SELECT
  grantee_name AS user_name,
  role,
  granted_by,
  created_on AS granted_at
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE deleted_on IS NULL
  AND (<USER_NAME_IS_NULL> OR grantee_name = '<USER_NAME>')
ORDER BY grantee_name, role;
```

### 3) Recently granted or revoked roles

```sql
SELECT
  grantee_name AS user_name,
  role,
  granted_by,
  created_on AS granted_at,
  deleted_on AS revoked_at,
  CASE WHEN deleted_on IS NULL THEN 'ACTIVE' ELSE 'REVOKED' END AS status
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE (
    created_on >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    OR deleted_on >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  )
  AND (<USER_NAME_IS_NULL> OR grantee_name = '<USER_NAME>')
  AND (<ROLE_NAME_IS_NULL> OR role = '<ROLE_NAME>')
ORDER BY COALESCE(deleted_on, created_on) DESC
LIMIT <LIMIT>;
```

### 4) Roles with broad privileges (OWNERSHIP, ALL PRIVILEGES)

> **Note:** `GRANTS_TO_ROLES` has up to **2 hours** latency.
> It tracks privilege-to-role grants. Active grants have `DELETED_ON IS NULL`.
> Does not contain grants on dropped objects.

```sql
SELECT
  grantee_name AS role_name,
  privilege,
  granted_on AS object_type,
  name AS object_name,
  table_catalog AS database_name,
  table_schema AS schema_name,
  grant_option,
  granted_by,
  created_on AS granted_at
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE deleted_on IS NULL
  AND privilege IN ('OWNERSHIP', 'ALL PRIVILEGES', 'ALL')
  AND granted_to = 'ACCOUNTROLE'
  AND (<ROLE_NAME_IS_NULL> OR grantee_name = '<ROLE_NAME>')
ORDER BY grantee_name, granted_on, name
LIMIT <LIMIT>;
```

### 5) Privilege count per role (find over-privileged roles)

```sql
SELECT
  grantee_name AS role_name,
  COUNT(*) AS total_grants,
  COUNT(DISTINCT privilege) AS distinct_privileges,
  COUNT(DISTINCT granted_on) AS distinct_object_types,
  SUM(CASE WHEN privilege = 'OWNERSHIP' THEN 1 ELSE 0 END) AS ownership_count,
  SUM(CASE WHEN grant_option = TRUE THEN 1 ELSE 0 END) AS with_grant_option_count
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE deleted_on IS NULL
  AND granted_to = 'ACCOUNTROLE'
GROUP BY grantee_name
ORDER BY total_grants DESC
LIMIT <LIMIT>;
```

### 6) Users with multiple high-privilege roles (separation of duties check)

```sql
SELECT
  grantee_name AS user_name,
  ARRAY_AGG(role) WITHIN GROUP (ORDER BY role) AS high_priv_roles,
  COUNT(*) AS role_count
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE deleted_on IS NULL
  AND role IN ('ACCOUNTADMIN', 'SYSADMIN', 'SECURITYADMIN', 'USERADMIN')
GROUP BY grantee_name
HAVING role_count > 1
ORDER BY role_count DESC;
```

### 7) (Optional) Recently changed privilege grants

```sql
SELECT
  grantee_name AS role_name,
  privilege,
  granted_on AS object_type,
  name AS object_name,
  granted_by,
  created_on AS granted_at,
  deleted_on AS revoked_at,
  CASE WHEN deleted_on IS NULL THEN 'ACTIVE' ELSE 'REVOKED' END AS status
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE (
    created_on >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
    OR deleted_on >= DATEADD('day', -<LOOKBACK_DAYS>, CURRENT_TIMESTAMP())
  )
  AND (<ROLE_NAME_IS_NULL> OR grantee_name = '<ROLE_NAME>')
ORDER BY COALESCE(deleted_on, created_on) DESC
LIMIT <LIMIT>;
```

## Output format
Return:
1) Users with high-privilege roles (ACCOUNTADMIN, SYSADMIN, etc.)
2) Separation of duties violations (users with multiple admin roles)
3) Over-privileged roles (ranked by total grants and ownership count)
4) Recent grant/revoke activity
5) Recommendations:
   - Minimize ACCOUNTADMIN assignments (principle of least privilege)
   - Separate SYSADMIN and SECURITYADMIN across different users
   - Review roles with excessive OWNERSHIP grants
   - Investigate recently revoked grants for potential security events
   - Use GRANT_OPTION sparingly (privilege escalation risk)
