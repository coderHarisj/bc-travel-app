---
trigger: always_on
---

# THE ARCHITECT’S DIRECTIVE  
## Enterprise-Grade Backend Standard (Concise • Enforceable • Non-Negotiable)
> **Prime Law:** Code must be **testable, replaceable, and deletable**.  
> If removing a feature causes fear or breaks unrelated logic, the architecture has failed.
## 1. Core Philosophy (SOLID as Law)
- **Single Responsibility:** One file = one job.
- **Open/Closed:** Extend via composition & DI, never modify core logic.
- **Liskov Substitution:** Implementations must be fully interchangeable.
- **Interface Segregation:** Small, precise interfaces only.
- **Dependency Inversion:** Business depends on abstractions, never frameworks.
## 2. Tech Stack Mastery
You operate strictly within the following ecosystem:
* **Language:** TypeScript
* **Runtime:** Node.js
* **Framework:** Express.js
* **Validation:** Yup
* **ORM:** Prisma
* **Database:** MySQL
* **Authentication:** JWT (Short-lived Access) + Refresh Tokens (Opaque/DB-backed)
* **Logging:** Custom DB-configurable structured logging (Zero-deployment updates)
## 3. Architecture & Structure (Layer-First)
/src
├─ /controllers    # feature.controller.ts
├─ /services       # feature.service.ts
├─ /interface    # feature.interface.ts
├─ /routes         # feature.routes.ts
├─ /schemas        # feature.schema.ts (Yup)
├─ /types          # feature.types.ts / DTOs
├─ /logger         # db-configurable logger
├─ /utils          # pure functions only
└─ /errors         # custom domain errors
**Rules**
- Separation by technical concern (Layered Architecture).
- Removing a feature requires cleanly deleting its associated files across all layer directories.
- No bypassing layers (e.g., Routes must not call Repositories directly).
## 4. Layer Responsibilities (Hard Boundaries)
### Controller (Boundary Only)
- Parse request
- Validate DTO/schema
- Call service
- Map response
Forbidden:
❌ Business logic  
❌ Prisma / DB calls  
❌ try/catch (Use Global Error Handler)  
### Service (Domain Only)
- Business rules
- Orchestration
- Pure & framework-agnostic
Forbidden:
❌ HTTP concepts (`req`/`res`)  
❌ Direct Prisma Client usage  
❌ Input validation (Yup belongs in Controller/Router)  
### Repository (Data Only)
- Queries & transactions
- Single data source (Prisma)
Forbidden:
❌ Business rules  
❌ API DTOs  
## 5. Validation & DTO Doctrine
- Validate **once**
- Validate **at boundary** (via Yup before reaching the Service)
- Types/DTOs define all external contracts
Forbidden:
❌ Raw request bodies passed to Services  
❌ Returning ORM entities directly to the client  
## 6. Dependency Flow
Router → Controller → Service → Repository
- Constructor injection only
- No circular dependencies
- Shared logic between services → extract to a shared domain service
Forbidden:
❌ `new PrismaClient()` instantiated inside services  
## 7. Utils & Helpers Strict Law
- `src/utils/` is exclusively for **pure functions** (e.g., math calculations, string formatting).
- If a function requires database access, environment variables, or complex business rules, it is a **Service**, not a utility.
Forbidden:
❌ Business logic in utils  
❌ God-object `index.ts` dumping grounds  
## 8. Error Strategy (Zero Chaos)
- No raw errors or stack traces leaked in production
- Typed domain errors only (e.g., `AppError`)
- Global Express error handler maps internal errors to proper HTTP responses
## 9. Security Is Default
Mandatory:
- Helmet
- Strict CORS
- Rate limiting
- Input sanitization (via Prisma parameterized queries)
- bcrypt password hashing
- Short-lived JWT + refresh tokens validated via DB
- No secrets in logs
Forbidden:
❌ Logging passwords/tokens  
❌ Trusting client input  
## 10. Database Discipline
- No queries outside repositories
- Prisma Transactions for multi-step logic
- Explicit failure handling
Forbidden:
❌ Partial writes  
❌ Hidden side effects  
## 11. Logging & Observability
- Structured JSON logs
- Request ID everywhere
- Log events, not noise
- Log configuration must be DB-driven and dynamically updateable without redeployment
- Never log PII, passwords, or full JWTs
## 12. Testability Standard
- Services testable completely without Express
- Repositories mocked (e.g., Mockito style / Prisma mock)
- Unit tests for all business logic
Forbidden:
❌ Testing Prisma behavior in service unit tests  
❌ Deep controller tests  
## 13. Clean Code Constraints
- Functions ≤ 20 lines
- Max nesting depth: 2
- No magic numbers
- Names > comments
- Consistent formatting
Forbidden:
❌ Dead code  
## FINAL LAW
> **Code is the documentation.** > If the intent isn’t obvious, refactor. No exceptions.