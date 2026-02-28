---
description: Generate Mermaid ER diagram from prisma.schema
---

ER Diagram Generation Workflow

> Trigger this workflow whenever prisma.schema is created or updated.

---

## STEP 1 — Read & Parse prisma.schema
`→ db-model-alignment.md` `→ db-identity-relationships.md`
- Read the full `prisma.schema` file.
- Extract: all models, fields (name + type + modifiers), enums, and `@relation` directives.
- **Exit:** Every model and relation accounted for. No models skipped.

## STEP 2 — Extract Relationship Map
`→ db-identity-relationships.md`
- For each `@relation`, determine: parent model, child model, FK field, and cardinality.
- Cardinality rules:
  - Optional field (`?`) on FK side → 1-M with nullable FK.
  - `@unique` on FK field → 1-1.
  - Join model with two FKs → M-M.
- **Exit:** Every relationship has a labeled type (1-1 / 1-M / M-M).

## STEP 3 — Generate Mermaid ER Diagram
`→ db-model-alignment.md` `→ db-identity-relationships.md`
- Output a valid `erDiagram` Mermaid block.
- **Include:** `_id`, `_key`, all business fields, FK columns, enums as string type.
- **Exclude:** audit fields (`createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `deletedAt`, `deletedBy`).
- Relationship notation:

| Cardinality | Mermaid Syntax  |
|-------------|-----------------|
| 1-1         | `||--||`        |
| 1-M         | `||--o{`        |
| M-M         | `}o--o{`        |

- Join tables MUST appear as explicit nodes — never collapsed into a direct M-M line.
- **Exit:** Mermaid block renders without errors. All entities and relationships visible.

## STEP 4 — Output
- Wrap the diagram in a fenced mermaid code block.
- Add a relationship summary table below the diagram:

| Entity A | Relationship | Entity B | Via              |
|----------|-------------|----------|------------------|
| User     | 1-M         | Booking  | booking.user_id  |

- **Exit:** Diagram + summary table delivered. Ready to paste into docs or PR.

---

## Hard Gates
- [ ] Every model in prisma.schema appears in the diagram
- [ ] Every `@relation` is represented with correct cardinality
- [ ] Join tables shown as explicit nodes
- [ ] Audit fields excluded for clarity
- [ ] Mermaid block is valid and renders correctly