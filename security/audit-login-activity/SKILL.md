---
name: audit-login-activity
description: Detect failed logins, brute-force attempts, MFA compliance gaps, and unusual login patterns using LOGIN_HISTORY.
---

# Audit Login Activity

## Goal
Quickly answer:
- Are there **failed login attempts** that indicate brute-force or credential issues?
- Which **IP addresses** are generating failed logins?
- Are users logging in **without MFA**?
- What **client types** (JDBC, Python, Snowsight, etc.) are being used?
- Are there logins from **new or unusual IPs**?

## Inputs to collect
- `lookback_hours` (optional, default 24)
- Optional: `user_name`
- Optional: `limit` (optional, default 100)

## Workflow

### 1) Failed logins grouped by user (brute-force detection)

> **Note:** `LOGIN_HISTORY` has up to **2 hours** latency.
> `IS_SUCCESS` is a VARCHAR column with values `'YES'` or `'NO'`.

```sql
SELECT
  user_name,
  COUNT(*) AS failed_attempts,
  MIN(event_timestamp) AS first_failure,
  MAX(event_timestamp) AS last_failure,
  ARRAY_AGG(DISTINCT client_ip) AS source_ips,
  ARRAY_AGG(DISTINCT error_code) AS error_codes,
  ANY_VALUE(error_message) AS sample_error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND is_success = 'NO'
  AND (<USER_NAME_IS_NULL> OR user_name = '<USER_NAME>')
GROUP BY user_name
ORDER BY failed_attempts DESC
LIMIT <LIMIT>;
```

### 2) Failed logins grouped by IP (suspicious IP detection)

```sql
SELECT
  client_ip,
  COUNT(*) AS failed_attempts,
  ARRAY_AGG(DISTINCT user_name) AS targeted_users,
  MIN(event_timestamp) AS first_failure,
  MAX(event_timestamp) AS last_failure,
  ANY_VALUE(error_message) AS sample_error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND is_success = 'NO'
GROUP BY client_ip
ORDER BY failed_attempts DESC
LIMIT <LIMIT>;
```

### 3) Users without MFA (successful logins only)
Identify users who logged in successfully without a second authentication factor.

```sql
SELECT
  user_name,
  COUNT(*) AS logins_without_mfa,
  ARRAY_AGG(DISTINCT first_authentication_factor) AS auth_methods,
  ARRAY_AGG(DISTINCT client_ip) AS source_ips,
  MAX(event_timestamp) AS last_login
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND is_success = 'YES'
  AND second_authentication_factor IS NULL
  AND (<USER_NAME_IS_NULL> OR user_name = '<USER_NAME>')
GROUP BY user_name
ORDER BY logins_without_mfa DESC
LIMIT <LIMIT>;
```

### 4) Login activity by client type
Understand which drivers/tools are used to connect.

```sql
SELECT
  reported_client_type,
  reported_client_version,
  COUNT(*) AS login_count,
  COUNT(DISTINCT user_name) AS distinct_users,
  SUM(CASE WHEN is_success = 'YES' THEN 1 ELSE 0 END) AS success_count,
  SUM(CASE WHEN is_success = 'NO' THEN 1 ELSE 0 END) AS failure_count
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
GROUP BY reported_client_type, reported_client_version
ORDER BY login_count DESC;
```

### 5) Successful login summary per user (recent activity overview)

```sql
SELECT
  user_name,
  COUNT(*) AS total_logins,
  COUNT(DISTINCT client_ip) AS distinct_ips,
  COUNT(DISTINCT reported_client_type) AS distinct_clients,
  MIN(event_timestamp) AS first_login,
  MAX(event_timestamp) AS last_login,
  SUM(CASE WHEN second_authentication_factor IS NOT NULL THEN 1 ELSE 0 END) AS mfa_logins,
  SUM(CASE WHEN second_authentication_factor IS NULL THEN 1 ELSE 0 END) AS non_mfa_logins
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
  AND is_success = 'YES'
  AND (<USER_NAME_IS_NULL> OR user_name = '<USER_NAME>')
GROUP BY user_name
ORDER BY total_logins DESC
LIMIT <LIMIT>;
```

### 6) (Optional) Logins from new IPs not seen in prior period
Compare current period IPs against a baseline to find newly observed source IPs.

```sql
WITH current_ips AS (
  SELECT DISTINCT client_ip, user_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
  WHERE event_timestamp >= DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
    AND is_success = 'YES'
),
baseline_ips AS (
  SELECT DISTINCT client_ip, user_name
  FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
  WHERE event_timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP())
    AND event_timestamp < DATEADD('hour', -<LOOKBACK_HOURS>, CURRENT_TIMESTAMP())
    AND is_success = 'YES'
)
SELECT
  c.user_name,
  c.client_ip AS new_ip
FROM current_ips c
LEFT JOIN baseline_ips b
  ON c.client_ip = b.client_ip
 AND c.user_name = b.user_name
WHERE b.client_ip IS NULL
ORDER BY c.user_name;
```

## Output format
Return:
1) Failed login summary (users with most failures, suspicious IPs)
2) MFA compliance gaps (users logging in without MFA)
3) Client type breakdown
4) New/unusual IPs (if detected)
5) Recommendations:
   - If a user has many failures: check credentials, reset password, or check for service account misconfiguration
   - If an IP targets multiple users: consider network policy to block it
   - If MFA adoption is low: enforce MFA via security policies
   - If new IPs appear for sensitive users: verify with the user
