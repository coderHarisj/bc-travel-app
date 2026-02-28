---
description: Generate core database schema from PRD
---

---
description: Schema design workflow — step-by-step with rule file references
alwaysApply: true
---

# Database Schema Design Workflow

> Follow steps in order. Each step must pass its exit criteria before proceeding.

---

## STEP 1 — Analyze Requirements
`→ db-model-alignment.md` `→ db-governance-audit.md` `→ db-performance-strategy.md`
- Extract functional requirements, top 10 query patterns, sensitive data categories, retention needs.
- **Exit:** Requirements + query patterns documented. Sensitive fields identified.

## STEP 2 — Identify Business Domains
`→ db-model-alignment.md` `→ db-identity-relationships.md`
- Group requirements into bounded domains. Define ownership. Flag cross-domain relationships.
- **Exit:** Domain list with ownership boundaries. No redundant domains.

## STEP 3 — Define Entities, Relationships & Identity
`→ db-identity-relationships.md` `→ db-integrity-controls.md` `→ db-model-alignment.md`
- Every entity must define: `<entity>_id BIGINT PK` (internal) + `<entity>_key UUID UNIQUE` (external/API).
- Every relationship must state: type (1-1 / 1-M / M-M), FK side, `ON DELETE` behavior.
- M-M → explicit join table named `<entity_a>_<entity_b>` (alphabetical) with composite PK.
- **Exit:** All entities have dual-column identity. All relationships typed. No circular dependencies.

## STEP 4 — Generate Prisma Models
`→ db-integrity-controls.md` `→ db-identity-relationships.md` `→ db-model-alignment.md`
- Use `BigInt @id @default(autoincrement())` for PK. Use `String @unique @default(uuid())` for key.
- No `Float` for money → use `Decimal`. No bare `String` for bounded values → use `enum`.
- All `@relation` directives must reference explicit FK fields. All `@@unique` constraints declared.
- **Exit:** `prisma validate` passes. No type violations. All relations explicit.

## STEP 5 — Apply Audit, Soft Delete & Indexes
`→ db-governance-audit.md` `→ db-performance-strategy.md` `→ db-integrity-controls.md`
- **Audit fields** (every model): `createdAt`, `updatedAt` (`@updatedAt`), `createdBy`, `updatedBy`.
- **Soft delete** (business-critical models): `deletedAt DateTime?`, `deletedBy BigInt?`.
- **Mandatory indexes:** every FK column, every status/enum column, `createdAt`, `deletedAt`.
- **Conditional indexes:** composite for multi-filter queries. Partial for soft-delete scopes.
- No index without a comment stating which query it serves. No redundant indexes.
- **Exit:** Audit fields on all models. Indexes named, justified, non-redundant.

## STEP 6 — Finalize Schema & Migration
`→ db-governance-audit.md` `→ db-evolution-strategy.md`
- Add header comment block to every model: purpose, owner, retention, sensitive fields, nullable register.
- Migration naming: `V<version>__<description>.sql`. Destructive changes require PR sign-off + backup.
- **Exit:** `prisma validate` clean. Every model documented. Migration versioned.

## STEP 7 — ER Diagram
`→ db-model-alignment.md` `→ db-identity-relationships.md`
- Generate from final schema. Must show all entities, cardinality labels, FK connections, join tables.
- **Exit:** Diagram attached to PR. All relationships labeled.

## STEP 8 — Schema Documentation
`→ All rule files`
- Document per entity: fields, relationships, indexes + query served, business rules (DB vs app-enforced), retention.
- **Exit:** Every entity documented. Business rules register complete with enforcement location.

---

## PR Hard Gates — All Must Pass Before Merge
- [ ] `prisma validate` clean
- [ ] Every model has `_id` (BIGINT) + `_key` (UUID) + 4 audit fields
- [ ] Every FK has `ON DELETE` declared + an index
- [ ] Every index named and justified
- [ ] No `Float` for money. No plain-text sensitive fields
- [ ] No offset pagination on tables projected >10k rows
- [ ] No circular dependencies
- [ ] ER diagram attached
- [ ] Migration versioned and named correctly