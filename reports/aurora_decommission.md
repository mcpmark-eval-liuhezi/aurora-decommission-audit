# Aurora Project Decommission Report

**Project:** Aurora delivery project
**Governing document:** Aurora Project Data Retention Policy 2026 (`data_retention_policy_2026.pdf`)
**Owner:** Platform engineering (decommission owner)
**Scope:** Snowflake environment teardown, retention rule application, decommission evidence, teardown checklist totals, and evidence retention.

---

## ⚠️ EXECUTION STATUS — SNOWFLAKE TEARDOWN NOT COMPLETED

The Snowflake portion of this decommission **could not be executed**. The sections below record the
policy-mandated design (what the environment and evidence log are required to look like), and this
section records what actually happened.

The Snowflake connection is failing at the account level for every operation attempted, including
read-only listing:

```
250001 (08001): Failed to connect to DB: TLRKMNK-HZ33053.snowflakecomputing.com:443.
Your free trial has ended and all of your virtual warehouses have been suspended.
Add billing information in the Snowflake web UI to continue using the full set of Snowflake features.
```

Separately, once a connection is restored the write tools will need their allow-list configuration
repaired — the `snowflake_write_query` / `snowflake_create_schemas` / `snowflake_drop_databases`
paths reject every database with:

```
Access denied: Database 'AURORA_OPS' is not in the allowed databases list: ['null']
```

**Impact:** The database `AURORA_OPS`, its four schemas, and the evidence table
`CORE.DECOMMISSION_LOG` were NOT created, loaded, or torn down. Nothing was dropped, so no data has
been lost. The account is in its original state. **This report must not be treated as confirmation
of a completed teardown.**

### Remediation steps

1. Restore Snowflake access (add billing information / reinstate the suspended trial account
   `TLRKMNK-HZ33053`).
2. Repair the MCP Snowflake allow-list so `AURORA_OPS` is permitted.
3. Re-run Section 8 (the exact SQL statements needed are included below).
4. Update this report to remove this status block.

---

## 1. Dropped schemas

Per policy clause **R1** — *"Drop STAGING and SANDBOX at project close."* — the temporary schemas are
designated for removal:

| Schema | Type | Clause applied |
|--------|------|----------------|
| `STAGING` | temporary | R1 |
| `SANDBOX` | temporary | R1 |

## 2. Surviving schemas

Per policy clause **R2** — *"Never drop CORE or ARCHIVE."* — the permanent schemas must survive the
teardown:

| Schema | Type | Clause applied |
|--------|------|----------------|
| `CORE` | permanent | R2 |
| `ARCHIVE` | permanent | R2 |

**Required end state:** the project database `AURORA_OPS` contains only the permanent schemas
(`CORE`, `ARCHIVE`).

## 3. Decommission evidence log

Per Section 4 of the policy, before any schema is dropped the evidence table
**`CORE.DECOMMISSION_LOG`** must be loaded with one row per dropped schema, recording the schema name
and the policy clause applied. The evidence table must remain queryable after teardown.

Required contents — **one row per dropped schema** (2 rows):

| SCHEMA_NAME | POLICY_CLAUSE |
|-------------|---------------|
| `STAGING`   | R1            |
| `SANDBOX`   | R1            |

Full qualification: `AURORA_OPS.CORE.DECOMMISSION_LOG`.

## 4. Teardown checklist totals (Appendix A)

Appendix A of the policy lists the teardown checklist, numbered **TC-01** upward.

- **Total number of checklist items: 60** (TC-01 through TC-60)
- **Number of items carrying the `[DROP]` flag: 12**

The 12 `[DROP]` items are: TC-02, TC-03, TC-10, TC-13, TC-15, TC-20, TC-25, TC-30, TC-34, TC-36,
TC-40, TC-47.

## 5. Evidence retention period

Per policy clause **R3** — *"Keep decommission evidence for 90 days."* — decommission evidence must be
retained for **90 days**.

## 6. Audit pack

- **Policy document:** `data_retention_policy_2026.pdf` (12 pages) in the shared workspace.
- **Retention Rules evidence page:** policy page 4 (Section 3), containing exactly:

```
3. RETENTION RULES
R1. Drop STAGING and SANDBOX at project close.
R2. Never drop CORE or ARCHIVE.
R3. Keep decommission evidence for 90 days.
```

  Extracted and content-verified. **Note:** it is currently saved at `/tmp/mcp_ws/retention_rules.pdf`
  rather than the intended `evidence/retention_rules.pdf`, because the `evidence/` subdirectory does
  not exist in the workspace and no available tool can create a directory. Move it to
  `evidence/retention_rules.pdf` once that directory is created.

## 7. Standard environment (Section 2)

The policy defines the standard project database as `AURORA_OPS`, containing exactly four schemas:
`CORE`, `ARCHIVE`, `STAGING`, `SANDBOX`. Per Section 2.1, auditors expect this full layout to exist
before any teardown begins. This environment was **not** created — see the execution status block.

## 8. Required Snowflake statements

To complete the teardown once access is restored, execute in this order (Section 2 / 2.1, then
Section 4, then Section 3):

```sql
-- 1. Recreate the standard environment (Section 2)
CREATE DATABASE IF NOT EXISTS AURORA_OPS;
CREATE SCHEMA IF NOT EXISTS AURORA_OPS.CORE;
CREATE SCHEMA IF NOT EXISTS AURORA_OPS.ARCHIVE;
CREATE SCHEMA IF NOT EXISTS AURORA_OPS.STAGING;
CREATE SCHEMA IF NOT EXISTS AURORA_OPS.SANDBOX;

-- 2. Load the evidence table BEFORE dropping anything (Section 4)
CREATE TABLE IF NOT EXISTS AURORA_OPS.CORE.DECOMMISSION_LOG (
    SCHEMA_NAME    VARCHAR,
    POLICY_CLAUSE  VARCHAR
);

INSERT INTO AURORA_OPS.CORE.DECOMMISSION_LOG (SCHEMA_NAME, POLICY_CLAUSE) VALUES
    ('STAGING',  'R1'),
    ('SANDBOX',  'R1');

-- 3. Apply the retention rules (Section 3, clause R1 only — R2 forbids touching CORE/ARCHIVE)
DROP SCHEMA AURORA_OPS.STAGING;
DROP SCHEMA AURORA_OPS.SANDBOX;

-- 4. Verify the end state
SELECT SCHEMA_NAME FROM AURORA_OPS.INFORMATION_SCHEMA.SCHEMATA
 WHERE SCHEMA_NAME IN ('CORE','ARCHIVE','STAGING','SANDBOX');
-- Expected: CORE, ARCHIVE only

SELECT COUNT(*) AS DROPPED_SCHEMA_ROWS FROM AURORA_OPS.CORE.DECOMMISSION_LOG;
-- Expected: 2
```

---

*Prepared under the Aurora Project Data Retention Policy 2026. Sections 1–5 state the
policy-mandated outcome; the execution status block records that the Snowflake changes could not be
applied.*
