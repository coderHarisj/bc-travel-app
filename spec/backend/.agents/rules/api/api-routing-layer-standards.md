---
trigger: always_on
---

# API Routing & Layer Boundaries (Hard Enforcement)
# 1. Dependency Flow (Non-Negotiable)
Allowed flow:
Routes → Controllers → Services → Repositories
No layer may be skipped.
No reverse dependency allowed.
No circular dependencies allowed.
If a layer is bypassed, architecture integrity is compromised.
# 2. Routing Standards
## 2.1 Route Design
- REST semantics strictly enforced
- Resource-oriented paths only
- HTTP verbs define action
- URLs define resources
Correct:
GET    /users
POST   /users
GET    /users/:id
PUT    /users/:id
DELETE /users/:id
Incorrect:
POST /createUser
GET  /getAllUsers
POST /users/delete
## 2.2 Versioning
- Versioning must be in URL prefix
- Example: /api/v1/users
- No header-based or query-based versioning
## 2.3 Route File Rules
- One route file per feature
- No inline logic
- No anonymous async handlers
- Routes must only:
  - Bind HTTP method
  - Attach controller
  - Apply middleware
Forbidden:
- Business logic
- DB access
- Response shaping
- Conditional branching
Routes are wiring only.
# 3. Controller Layer (Boundary Only)
## Responsibilities
- Parse request
- Validate input (after middleware validation)
- Call service
- Map response to DTO
Controllers are thin adapters between HTTP and domain logic.
## Forbidden
- Business logic
- Database access
- Direct Prisma usage
- try/catch blocks
- Conditional branching based on domain rules
Controllers must be disposable and easily replaceable.
# 4. Service Layer (Domain Only)
## Responsibilities
- Business rules
- Orchestration
- Decision making
- Calling repositories
Services define application behavior.
## Forbidden
- HTTP objects (req, res, headers)
- ORM entities exposed externally
- Validation logic (belongs at boundary)
- Response formatting
Services must be:
- Framework-agnostic
- Pure from transport concerns
- Fully unit testable
# 5. Repository Layer (Data Only)
## Responsibilities
- Data persistence
- Query composition
- Transactions
- Data source interaction
Repositories are the only layer allowed to talk to Prisma.
## Forbidden
- Business decisions
- DTO mapping
- Cross-repository orchestration logic
- HTTP awareness
# 6. Architectural Enforcement Rules
## 6.1 Constructor Injection Only
Dependencies must be injected.
No internal instantiation of services or repositories.
Forbidden:
new PrismaClient() inside services  
new Repository() inside controllers  
## 6.2 Feature Deletion Rule
If deleting a feature:
- Remove its route file
- Remove its controller
- Remove its service
- Remove its repository
- No other feature must require modification
If removal causes breakage elsewhere, layering has failed.
# 7. Design Integrity Test
Ask these questions:
- Can the service run without Express?
- Can the repository be swapped without service changes?
- Can the controller be rewritten without touching business logic?

If any answer is “no”, refactor immediately.