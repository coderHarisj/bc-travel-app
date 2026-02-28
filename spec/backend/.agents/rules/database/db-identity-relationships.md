---
trigger: always_on
---

---
description: Enforce identity and relationship modeling standards
alwaysApply: true
---

# Identity & Relationship Design Rules

## 1. Primary Key — Mandatory on Every Table
Every table MUST have an explicitly defined BIGINT primary key. No implicit, UUID-only, or composite-only PKs unless justified.

| Scenario                        | Required PK Approach                                        |
|---------------------------------|-------------------------------------------------------------|
| All standard entities           | BIGINT auto-increment — named `<table_singular>_id`         |
| Join tables (M-M)               | Composite PK of both FK BIGINT columns                      |
| Lookup/reference tables         | BIGINT sequence or short string code                        |

- PK column MUST follow the naming convention `<table_singular>_id`.
  - `bookings` table → `booking_id BIGINT`
  - `users` table → `user_id BIGINT`
  - `episodes` table → `episode_id BIGINT`
- PK must be declared with `PRIMARY KEY` constraint explicitly. Never assumed by convention.
- BIGINT is mandatory — INT (32-bit) is prohibited as a PK due to overflow risk at scale.

## 2. External Key (UUID) — Mandatory on Every Public-Facing Entity
Every table whose records are referenced in API responses, URLs, or external systems MUST have a separate `_key` column storing a UUID v4.

| Column            | Type               | Constraint               | Default                   |
|-------------------|--------------------|--------------------------|---------------------------|
| `booking_id`      | BIGINT             | PRIMARY KEY, NOT NULL    | Auto-increment (SERIAL)   |
| `booking_key`     | UUID               | UNIQUE, NOT NULL         | `gen_random_uuid()`       |

Rules for the `_key` column:
- Naming convention: `<table_singular>_key` (e.g. `booking_key`, `user_key`, `episode_key`).
- MUST be `UNIQUE` and `NOT NULL` — enforced at DB level, not just application level.
- MUST have a dedicated index: `CREATE UNIQUE INDEX idx_bookings_booking_key ON bookings(booking_key)`.
- Generated at insert time via `gen_random_uuid()` — never supplied by the client.
- MUST be immutable after insert — no UPDATE allowed on `_key` columns ever.
- The `_key` UUID is the ONLY identifier ever exposed in:
  - API responses
  - URL path parameters (`/bookings/{booking_key}`)
  - Webhooks and external event payloads
- The internal `_id` BIGINT MUST NEVER appear in any API response, URL, or external system.

## 3. ID Generation Strategy — Must Be Declared Per Table
Every table's schema comment MUST state which strategy is used for both columns.
```sql
-- TABLE: bookings
-- PK STRATEGY: BIGINT auto-increment (internal use only)
-- KEY STRATEGY: UUID v4 via gen_random_uuid() (external/API use)
```

| Column Type  | Strategy           | Exposure          |
|--------------|--------------------|-------------------|
| `_id`        | BIGINT SERIAL      | Internal DB only  |
| `_key`       | UUID v4 (random)   | API / external    |

- No other ID strategies are permitted without explicit approval and schema comment justification.
- Mixed strategies across related tables must be flagged in a schema review.

## 4. Relationship Types — Must Be Explicitly Modeled

### One-to-One (1-1)
- FK lives on the weaker/dependent entity, not the primary one.
- FK column references the BIGINT `_id` of the parent — never the `_key`.
- FK column must have a `UNIQUE` constraint to enforce the 1-1.
- Example: `user_profiles.user_id BIGINT UNIQUE REFERENCES users(user_id)`

### One-to-Many (1-M)
- FK lives on the "many" side — never store an array of IDs on the "one" side.
- FK column references the BIGINT `_id` of the parent.
- Example: `orders.user_id BIGINT REFERENCES users(user_id)` — NOT `users.order_ids BIGINT[]`
- Array of FK references on a parent is a schema violation in relational DBs.

### Many-to-Many (M-M)
- MUST use an explicit join table. No array columns for M-M in relational DBs.
- Join table naming convention: `<entity_a>_<entity_b>` in alphabetical order (e.g. `episode_tags`, `role_permissions`).
- Join table MUST include:
  - Composite PK of both FK BIGINT columns.
  - `created_at TIMESTAMPTZ` — when the relationship was formed.
  - `created_by BIGINT` — FK → `users(user_id)`, who created the relationship.
- If the relationship carries its own data (e.g. a user's role in a project with an assigned date), the join table is promoted to a full entity with its own `_id` BIGINT PK and `_key` UUID.

## 5. Foreign Keys — Explicit Declaration Required
- Every FK MUST reference the BIGINT `_id` column of the parent — never the `_key` UUID.
- The `_key` UUID is for external lookup only — it is never used as a FK target.
- FK column naming convention: `<referenced_table_singular>_id` (e.g. `user_id`, `episode_id`, `plan_id`).
- Every FK MUST be declared with an explicit `REFERENCES` constraint. Implied FKs in comments only are not acceptable.
- Every FK MUST define an `ON DELETE` behavior. Silence is not acceptable.

| Scenario                                        | Required ON DELETE behavior    |
|-------------------------------------------------|--------------------------------|
| Child record is meaningless without parent      | `CASCADE`                      |
| Child record should be preserved                | `SET NULL` (FK must be nullable)|
| Deletion must be blocked if children exist      | `RESTRICT`                     |
| Audit/log records referencing deleted rows      | `SET NULL` or soft delete      |

- Every FK column MUST have a corresponding index to prevent full table scans on joins.

## 6. Circular Dependencies — Explicitly Prohibited
- No two tables may have FK references pointing to each other simultaneously as hard `NOT NULL` constraints.
- Example violation: `users.current_subscription_id → subscriptions(subscription_id)` AND `subscriptions.user_id → users(user_id)` — both as NOT NULL FKs creates an unresolvable insert order.
- Resolution strategies:
  - Make one FK nullable and populate it after both records are inserted.
  - Use a status/flag column instead of a back-reference FK.
  - Introduce an intermediary entity to break the cycle.
- A circular dependency found in schema review is an automatic rejection until resolved.

## 7. Query Pattern Alignment — Design Must Justify Structure
Before finalizing any relationship design, the following must be documented in schema comments:

- What are the top 5 queries this schema will serve?
- Which fields appear in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY`?
- Are those fields indexed? (Cross-reference with performance rules.)

Relationship design violations triggered by query patterns:
- Joining more than 4 tables to retrieve a single business object → consider denormalization or a materialized view.
- The same JOIN repeated across 5+ queries → relationship structure needs revisiting or a view must be created.

## 8. Redundant Relational Duplication — Prohibited
- The same relationship MUST NOT be expressed in more than one way in the schema.
- Example violation: storing both `orders.user_id` (FK) and `orders.user_email` (denormalized copy) — pick one unless caching is explicitly justified.
- If denormalization is intentional for read performance, it MUST be:
  - Documented in a schema comment explaining why.
  - Kept in sync via a DB trigger or application-level contract.
  - Flagged for review in the next schema audit cycle.