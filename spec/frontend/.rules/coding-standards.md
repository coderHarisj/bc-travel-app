# Coding Standards & Principles
Project: Tourism Management System

## Core Engineering Principles

### DRY (Don't Repeat Yourself)
- Shared logic must be extracted to:
  - shared/hooks
  - shared/utils
  - shared/components
- No duplicate validation logic across features
- No duplicated API transformation logic

### KISS (Keep It Simple)
- Avoid unnecessary abstraction
- Avoid over-engineering patterns
- Prefer readable over clever code

### SRP (Single Responsibility Principle)
- Each component handles one concern only
- Pages orchestrate only
- Services communicate with API only
- Slices manage state only

### OCP (Open-Closed Principle)
- Extend via composition, not modification
- Use hooks to extend behavior
- Avoid editing core components to add feature logic

---

## Component Structure Rules

Each component must:
- Have a clear purpose
- Avoid internal business logic
- Not exceed reasonable size (split if large)
- Separate UI from logic using custom hooks when needed

---

## Naming Conventions

| Element | Convention | Example |
|----------|------------|----------|
| Components | PascalCase | LeadForm.jsx |
| Hooks | use + camelCase | useDebounce.js |
| Services | camelCase + Service | leadService.js |
| Redux Slices | camelCase + Slice | leadsSlice.js |
| Constants | SCREAMING_SNAKE_CASE | API_BASE_URL |

---

## File Organization Rules

- One component per file
- No mixed responsibilities in same file
- Tests must mirror source structure

---

## Forbidden Patterns

- Business logic inside JSX
- Hardcoded strings (extract constants if reused)
- Direct mutation of Redux state outside reducers
- Large monolithic components