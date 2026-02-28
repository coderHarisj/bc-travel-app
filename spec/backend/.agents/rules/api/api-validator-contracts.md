---
trigger: always_on
---

# API Validation & Contracts

## Validation Law
Validate once. At the boundary. No exceptions.
## Rules
- Yup schemas only
- Validation occurs before controller execution
- Services never receive raw request bodies
## DTO Discipline
- Every request and response has an explicit DTO
- DTOs define the public API contract
- ORM entities never escape the service layer
Forbidden:
- Passing req.body directly to services
- Returning Prisma models in responses