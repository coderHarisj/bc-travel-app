---
trigger: always_on
---

---
description: Enforce data integrity and validation
alwaysApply: true
---

# Data Integrity & Validation Rules

## 1. Data Types — Correct Type is Mandatory, No Lazy Defaults
Every column MUST use the most precise type that fits the data. Using `VARCHAR` or `TEXT` as a catch-all is a schema violation.

| Data Category         | Required Type               | Prohibited Types                  |
|-----------------------|-----------------------------|-----------------------------------|
| Internal PK           | `BIGINT`                    | `INT`, `TEXT`, `VARCHAR`          |
| External key          | `UUID`                      | `VARCHAR`, `TEXT`                 |
| Short text (≤ 100)    | `VARCHAR(n)` with explicit n| `TEXT` for bounded strings        |
| Long text (unbounded) | `TEXT`                      | `VARCHAR(10000)` as a workaround  |
| Whole numbers         | `BIGINT` or `INT`           | `DECIMAL`, `FLOAT` for counts     |
| Money / currency      | `NUMERIC(19,4)`             | `FLOAT`, `DOUBLE`, `INT`          |
| Boolean flags         | `BOOLEAN`                   | `INT` (0/1), `CHAR` ('Y'/'N')     |
| Timestamps            | `TIMESTAMPTZ` (UTC)         | `DATETIME`, `VARCHAR` for dates   |
| Date only             | `DATE`                      | `TIMESTAMPTZ` for date-only data  |
| Status / category     | `ENUM` or `VARCHAR` + CHECK | `INT` codes without a lookup      |
| JSON / flexible data  | `JSONB`                     | `TEXT` storing raw JSON strings   |

- `FLOAT` and `DOUBLE` are PROHIBITED for any financial or precision-sensitive column — they introduce rounding errors.
- All timestamps MUST be `TIMESTAMPTZ` (timezone-aware). `TIMESTAMP WITHOUT TIME ZONE` is prohibited.
- `VARCHAR` without a length limit (i.e. `VARCHAR` with no `n`) is treated the same as `TEXT` — use `TEXT` explicitly if unbounded.

## 2. NOT NULL — Default Stance is Non-Nullable
Every column is `NOT NULL` by default. Nullable columns require explicit justification documented in a schema comment.

- Acceptable justifications for `NULL`:
  - The value is genuinely unknown at insert time and will be populated later (e.g. `completed_at`, `deleted_at`).
  - The FK relationship is optional by business logic (e.g. a booking may or may not have a promo code).
  - The field is part of a soft-delete or audit pattern.
- Unacceptable justifications:
  - "We might need it later" — design for what exists now.
  - "The ORM defaults to nullable" — ORM defaults do not override schema rules.
  - No comment at all — undocumented nullable columns are a schema violation.

Every nullable column MUST have this comment pattern:
```sql
cancelled_at TIMESTAMPTZ NULL, -- NULLABLE: populated only when booking is cancelled
```

## 3. Unique Constraints — Applied Wherever Business Requires Uniqueness
A `UNIQUE` constraint MUST be applied at the DB level for any field or combination of fields that the business treats as unique. Application-layer uniqueness checks alone are not sufficient.

| Scenario                                      | Required Constraint                                      |
|-----------------------------------------------|----------------------------------------------------------|
| External key column (`_key`)                  | `UNIQUE` on `<table>_key`                                |
| User email                                    | `UNIQUE` on `users(email)`                               |
| Username / handle                             | `UNIQUE` on `users(username)`                            |
| A user can only have one active subscription  | `UNIQUE` on `subscriptions(user_id)` with partial index  |
| A user can only book a slot once              | `UNIQUE` on `bookings(user_id, slot_id)`                 |

- Composite unique constraints are allowed and encouraged for business-rule uniqueness.
- Partial unique indexes MUST be used when uniqueness applies only under a condition:
```sql
-- A user can only have one active subscription at a time
CREATE UNIQUE INDEX idx_subscriptions_active_user
ON subscriptions(user_id)
WHERE deleted_at IS NULL;
```

## 4. ENUM & CHECK Constraints — Required for Bounded Value Sets
Any column with a fixed, known set of valid values MUST use either a DB `ENUM` type or a `CHECK` constraint. Storing arbitrary strings for status fields is a violation.

### When to use ENUM:
- The value set is stable and rarely changes (e.g. `gender`, `day_of_week`).
- Values are used across multiple queries and need DB-level type safety.
```sql
CREATE TYPE booking_status AS ENUM ('pending', 'confirmed', 'cancelled', 'completed');
ALTER TABLE bookings ADD COLUMN status booking_status NOT NULL DEFAULT 'pending';
```

### When to use CHECK constraint:
- The value set may grow occasionally and adding an ENUM value requires a migration.
- The constraint is numeric or expression-based.
```sql
-- Status as VARCHAR + CHECK (easier to extend)
status VARCHAR(20) NOT NULL DEFAULT 'pending'
  CHECK (status IN ('pending', 'confirmed', 'cancelled', 'completed')),

-- Numeric range check
rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),

-- Price must be positive
amount NUMERIC(19,4) NOT NULL CHECK (amount > 0),

-- End date must be after start date
CHECK (end_date > start_date)
```

- Every ENUM or CHECK value MUST be documented with what it means, not just listed:
```sql
-- status values:
-- 'pending'   → booking created, awaiting payment
-- 'confirmed' → payment received, slot reserved
-- 'cancelled' → cancelled by user or system
-- 'completed' → session delivered
```

## 5. Business Rules — DB-Level Enforcement for Critical Rules
Business rules that are critical to data correctness MUST be enforced at the DB level, not left solely to the application layer.

| Rule Type                          | Enforcement Mechanism              |
|------------------------------------|------------------------------------|
| Value must be in a valid set       | `ENUM` or `CHECK` constraint       |
| Field must be unique               | `UNIQUE` constraint or index       |
| Field must be positive             | `CHECK (column > 0)`               |
| Two dates must be in order         | `CHECK (end_date > start_date)`    |
| Referential integrity              | `FOREIGN KEY` with `ON DELETE`     |
| One active record per user         | Partial `UNIQUE` index             |
| Computed field must stay in sync   | DB trigger or generated column     |

- Application-level validation (e.g. API request validation) is required in addition to DB constraints, not instead of them.
- If a business rule cannot be expressed as a DB constraint, it MUST be documented in the schema comment with the application layer location where it is enforced:
```sql
-- BUSINESS RULE: A user cannot book more than 3 active sessions simultaneously.
-- Enforced at: BookingService.validateSessionLimit() — not expressible as a DB constraint.
```

## 6. Nullable Field Register — Document Every Nullable Column
A schema MUST maintain a nullable field register either as a comment block or a separate doc entry listing every nullable column and its justification:
```sql
-- NULLABLE FIELD REGISTER: bookings
-- cancelled_at    → set when booking is cancelled, NULL = not cancelled
-- promo_code_id   → optional FK, NULL = no promo applied
-- completed_at    → set after session delivered, NULL = not yet completed
-- notes           → optional user-provided text, NULL = no notes added
```

- Any nullable column NOT listed in the register is a schema violation.
- The register must be updated as part of every migration that adds or removes a nullable column.

## 7. Validation Logic Documentation — Required Format
Every non-obvious constraint or validation rule MUST be documented inline:
```sql
-- Single field constraint
email VARCHAR(255) NOT NULL
  CHECK (email ~* '^[^@]+@[^@]+\.[^@]+$'), -- must be valid email format

-- Cross-column constraint
CONSTRAINT chk_booking_dates CHECK (end_date > start_date), -- end must be after start

-- Application-enforced rule (not in DB)
-- RULE: Maximum 3 concurrent active bookings per user
-- ENFORCED AT: BookingService.create() → validateConcurrentBookingLimit()
```

- Validation rules enforced only at the application layer MUST reference the exact service/function/module where they live.
- Schema documentation must distinguish clearly between DB-enforced and app-enforced rules so reviewers know where to look.