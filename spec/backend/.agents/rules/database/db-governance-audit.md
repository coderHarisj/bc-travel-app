---
trigger: always_on
---

---
description: Enforce governance and audit standards
alwaysApply: true
---

# Governance & Maintainability Rules

1. Include:
   - created_at
   - created_by
   - updated_at
   - updated_by
2. Define soft delete strategy:
   - deleted_at 
3. Add audit trail fields if required.
4. Identify sensitive fields (password, tokens).
5. Define retention strategy.
6. Schema must be clearly documented---
description: Enforce governance and audit standards
alwaysApply: true
---

# Governance & Maintainability Rules

## 1. Timestamp Fields — Mandatory on Every Table/Collection
Every table or collection MUST include these exact fields. No exceptions.

| Field        | Type                        | Default              | Nullable |
|--------------|-----------------------------|----------------------|----------|
| created_at   | TIMESTAMPTZ (UTC)           | CURRENT_TIMESTAMP    | NOT NULL |
| updated_at   | TIMESTAMPTZ (UTC)           | CURRENT_TIMESTAMP    | NOT NULL |
| created_by   | UUID (FK → users.id)        | set at app layer     | NOT NULL |
| updated_by   | UUID (FK → users.id)        | set at app layer     | NOT NULL |

- `updated_at` MUST be auto-updated via a DB trigger or ORM hook — manual updates are not acceptable.
- `created_at` and `created_by` MUST be immutable after insert — no UPDATE allowed on these columns.
- Timestamps MUST be stored in UTC. No local timezone storage.

## 2. Soft Delete — Required for Any Business-Critical Entity
If a record must not be permanently destroyed (users, orders, content, transactions), soft delete is mandatory.

| Field        | Type            | Default | Nullable |
|--------------|-----------------|---------|----------|
| deleted_at   | TIMESTAMPTZ     | NULL    | YES      |
| deleted_by   | UUID (FK → users.id) | NULL | YES     |

- A record is considered deleted when `deleted_at IS NOT NULL`.
- All queries on soft-deleted tables MUST include `WHERE deleted_at IS NULL` by default.
- If your ORM supports global scopes (e.g. Eloquent, Django managers), enforce this at the model layer, not ad-hoc in every query.
- Hard delete is ONLY permitted on: temporary/session records, logs older than retention window, or explicitly non-critical lookup tables.

## 3. Audit Trail — Required for Sensitive or Regulated Actions
For any table involving money, permissions, authentication, or user-generated content, an audit log entry MUST be created on every INSERT, UPDATE, and DELETE.

Audit log table must capture:

| Field        | Type          | Notes                            |
|--------------|---------------|----------------------------------|
| id           | UUID          | PK                               |
| table_name   | VARCHAR(100)  | Target table                     |
| record_id    | UUID          | PK of affected row               |
| action       | ENUM          | 'INSERT', 'UPDATE', 'DELETE'     |
| old_value    | JSONB         | State before change              |
| new_value    | JSONB         | State after change               |
| changed_by   | UUID          | FK → users.id                    |
| changed_at   | TIMESTAMPTZ   | UTC timestamp                    |

- Audit logs are APPEND-ONLY — no updates or deletes on audit_log table ever.
- Audit entries must be written in the same transaction as the change.

## 4. Sensitive Field Handling
Any field containing sensitive data MUST be flagged in the schema comment and handled as follows:

| Field Type        | Examples                   | Required Handling                          |
|-------------------|----------------------------|--------------------------------------------|
| Credentials       | password, pin              | Hashed only (bcrypt/argon2). Never plain.  |
| Tokens            | refresh_token, api_key     | Hashed at rest. Never logged.              |
| PII               | email, phone, national_id  | Marked with comment `-- PII`. Encrypt if regulated. |
| Financial         | card_number, account_no    | Never stored raw. Tokenize via vault/PSP.  |

- No sensitive field may appear in an index that stores the raw value.
- Sensitive fields must never be returned in a `SELECT *` — use explicit column lists.

## 5. Data Retention Rules
Each table must declare its retention policy in a schema comment block:
```sql
-- RETENTION: 7 years (regulatory) | Purge via: scheduled job | Soft delete first
```

Minimum expectations:
- User activity logs → retain 90 days, then archive or purge.
- Financial/transaction records → retain 7 years minimum.
- Session/token records → purge on expiry + 24 hours max.
- Deleted records (soft) → hard purge after retention window via a scheduled job, not manual SQL.

## 6. Schema Documentation — Required Format
Every table MUST have a header comment block in this format:
```sql
-- TABLE: orders
-- PURPOSE: Stores customer purchase transactions
-- OWNER: payments-team
-- RETENTION: 7 years
-- SENSITIVE FIELDS: payment_reference
-- LAST REVIEWED: YYYY-MM-DD
```

- Every non-obvious column MUST have an inline comment explaining its purpose.
- Enum values must document what each value means, not just list them.

## 7. Migration Versioning — Zero Tolerance for Unversioned Changes
- Every schema change MUST go through a versioned migration file. Direct `ALTER TABLE` in production is forbidden.
- Migration files must be named: `V<version>__<description>.sql` (e.g. `V012__add_deleted_at_to_users.sql`).
- Migrations must be: forward-only in production. Rollback scripts are written but stored separately.
- Destructive migrations (DROP COLUMN, DROP TABLE, data type change) require:
  - A PR review sign-off.
  - A backup confirmation before execution.
  - A deprecation period of at least 1 release cycle before the drop..
