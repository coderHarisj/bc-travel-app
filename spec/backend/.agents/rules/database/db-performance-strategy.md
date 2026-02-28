---
trigger: always_on
---

---
description: Enforce performance and scalability strategy
alwaysApply: true
---

# Performance & Scalability Rules

## 1. Indexing Strategy — Mandatory for All Query-Critical Fields
Indexes MUST be explicitly defined in the schema. Relying on the DB to "figure it out" is not acceptable.

### Mandatory Indexes — No Exceptions
Every table MUST have indexes on the following without requiring justification:

| Column Type                        | Index Type         | Naming Convention                          |
|------------------------------------|--------------------|--------------------------------------------|
| Primary Key (`_id`)                | Auto (PK)          | Handled by DB                              |
| External Key (`_key` UUID)         | `UNIQUE INDEX`     | `idx_<table>_<col>_key`                    |
| Every Foreign Key column           | `INDEX`            | `idx_<table>_<fk_col>`                     |
| `deleted_at` (soft delete)         | Partial `INDEX`    | `idx_<table>_deleted_at`                   |
| `created_at` (if ordered queries)  | `INDEX`            | `idx_<table>_created_at`                   |
| `status` / `state` columns         | `INDEX`            | `idx_<table>_status`                       |

### Conditional Indexes — Required When Query Pattern Demands
These MUST be added when the corresponding query pattern exists:

| Query Pattern                                 | Index Required                                      |
|-----------------------------------------------|-----------------------------------------------------|
| `WHERE email = ?`                             | `UNIQUE INDEX` on `email`                           |
| `WHERE user_id = ? ORDER BY created_at DESC`  | Composite: `(user_id, created_at DESC)`             |
| `WHERE status = 'active' AND deleted_at IS NULL` | Partial composite index with `WHERE deleted_at IS NULL` |
| `WHERE LOWER(email) = ?`                      | Functional index: `LOWER(email)`                    |
| Full text search on a column                  | `GIN` index on `TSVECTOR` column                    |
| `JSONB` field lookups                         | `GIN` index on the `JSONB` column                   |

### Index Naming Convention — Strictly Enforced
All indexes MUST follow this naming pattern — unnamed or auto-named indexes are a violation:
```sql
-- Single column
CREATE INDEX idx_bookings_user_id ON bookings(user_id);

-- Composite
CREATE INDEX idx_bookings_user_status ON bookings(user_id, status);

-- Partial
CREATE INDEX idx_bookings_active ON bookings(user_id, created_at DESC)
WHERE deleted_at IS NULL;

-- Unique
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Functional
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

## 2. Over-Indexing — Explicitly Prohibited
Every index has a write cost. Indexes must be justified, not sprinkled defensively.

Rules to prevent over-indexing:
- A table with fewer than 10,000 projected rows does NOT need more than its mandatory indexes — sequential scans are faster at this scale.
- No two indexes on the same table may be redundant. Example violation: having both `idx_bookings_user_id` and a composite `idx_bookings_user_id_status` — the single-column index is now redundant if the composite covers it.
- Index audit MUST be performed at schema review: for every index, state which specific query it serves.
- Indexes on columns with very low cardinality (e.g. `is_deleted BOOLEAN`, `gender ENUM(2 values)`) are prohibited unless used in a partial index with a high-cardinality filter.

Index justification format required in schema comments:
```sql
-- INDEX: idx_bookings_user_status
-- SERVES QUERY: Fetch all active bookings for a user on the dashboard
-- QUERY PATTERN: WHERE user_id = ? AND status = 'confirmed' AND deleted_at IS NULL
-- ESTIMATED SELECTIVITY: High (user_id narrows to ~10 rows avg)
```

## 3. Normalization & Denormalization — Must Be Justified
Neither full normalization nor denormalization is the default. The choice must be deliberate and documented.

### Normalize when:
- Data is write-heavy and consistency is critical (e.g. user profile updates must reflect everywhere instantly).
- The duplicated data would create update anomalies (changing a value in one place must not leave stale copies elsewhere).
- The relationship is 1-M or M-M and the child data varies per record.

### Denormalize when:
- A query joins 3+ tables to produce a frequently read result (e.g. a booking card showing user name, episode title, plan name).
- The denormalized field is read 100x more than it is written.
- The source value is effectively immutable after creation (e.g. `episode_title` on a `bookings` snapshot).

### Denormalization rules if applied:
- The denormalized column MUST be clearly marked in schema comments:
```sql
-- DENORMALIZED: copied from episodes.title at booking creation time
-- SOURCE: episodes.title
-- SYNC STRATEGY: immutable snapshot — not updated after insert
-- JUSTIFIED BY: avoid JOIN on every booking list query
episode_title VARCHAR(255) NOT NULL,
```
- If the value is NOT immutable, a sync mechanism MUST be defined (trigger, event, job) — "we'll handle it in the app" is not acceptable.

## 4. Read vs Write Pattern — Must Be Declared Per Table
Every table MUST declare its expected access pattern in its schema comment block:
```sql
-- ACCESS PATTERN: Read-heavy (95% reads, 5% writes)
-- READ QUERIES: list bookings by user, fetch booking detail, dashboard summary
-- WRITE QUERIES: create booking, update status, soft delete
-- PEAK LOAD: booking list called on every dashboard load (~500 req/min projected)
```

Design decisions driven by access pattern:

| Pattern            | Design Implications                                                   |
|--------------------|-----------------------------------------------------------------------|
| Read-heavy         | More indexes allowed, denormalization justified, consider caching layer|
| Write-heavy        | Minimize indexes, avoid triggers, consider queue-based writes          |
| Mixed              | Separate read/write models if CQRS is feasible, else index selectively |
| Append-only        | Partition by time, no updates, compress old partitions                 |

## 5. Pagination — Cursor-Based is the Standard
Offset-based pagination (`LIMIT x OFFSET y`) is prohibited on any table expected to exceed 10,000 rows.

### Required: Cursor-based pagination
```sql
-- Cursor pagination pattern using BIGINT PK
SELECT * FROM bookings
WHERE user_id = ?
  AND booking_id < :last_seen_id   -- cursor from previous page
  AND deleted_at IS NULL
ORDER BY booking_id DESC
LIMIT 20;
```

Rules:
- The cursor column MUST be indexed — `booking_id` (PK) qualifies automatically.
- Cursor value MUST be the internal BIGINT `_id`, never the UUID `_key` — integer comparison is faster.
- The cursor MUST be opaque to the client — encode it before sending (e.g. base64 of the BIGINT value) so clients cannot manipulate it.
- If sorting by a non-PK column (e.g. `created_at`), use a composite cursor: `(created_at, booking_id)` to handle ties.
- Offset pagination is ONLY acceptable for: admin panels with small bounded datasets, export jobs where page position doesn't shift mid-export.

## 6. Partitioning — Required for Tables Projected to Exceed 50M Rows
Tables projected to exceed 50 million rows within 2 years MUST define a partitioning strategy at schema design time, not as a retrofit.

### Partitioning decision table:

| Data Characteristic                  | Partition Strategy              | Example                                      |
|--------------------------------------|---------------------------------|----------------------------------------------|
| Time-series / append-heavy           | Range partition by `created_at` | Monthly partitions on `audit_logs`           |
| Tenant-based multi-tenancy           | List partition by `tenant_id`   | Partition `bookings` by `tenant_id`          |
| Evenly distributed high-volume       | Hash partition by PK            | Hash partition `events` by `event_id`        |
| Hybrid (time + tenant)               | Composite partition             | Range by month, sub-partition by `tenant_id` |

Partitioning declaration required in schema comment:
```sql
-- PARTITIONING: Range by created_at (monthly)
-- PROJECTED VOLUME: ~5M rows/month
-- PARTITION RETENTION: 12 months active, archive beyond that
-- TOOLING: pg_partman for automated partition creation
```

- Indexes on partitioned tables MUST be created on each partition — global indexes on partitioned tables have restrictions, document which approach is used.
- Partition pruning MUST be verified: queries on partitioned tables MUST include the partition key in the `WHERE` clause, otherwise all partitions are scanned.

## 7. Scalability Checklist — Required Sign-Off Before Schema Approval
Before a schema is approved, the following must be answered and documented:
```
SCALABILITY SIGN-OFF: <table_name>
------------------------------------------------------
[ ] Projected row count at 6 months / 1 year / 3 years estimated
[ ] Mandatory indexes defined and named
[ ] Every index justified with the query it serves
[ ] Offset pagination removed or justified
[ ] Partitioning assessed (required if >50M rows projected)
[ ] Read/write access pattern declared
[ ] Denormalized fields documented with sync strategy
[ ] Over-indexing audit completed (no redundant indexes)
[ ] FK indexes confirmed (every FK column has an index)
```

- This checklist MUST be included as a comment block in the migration file for any new table.
- Schema review is blocked until all items are checked off.