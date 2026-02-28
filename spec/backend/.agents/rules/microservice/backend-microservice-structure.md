---
trigger: always_on
---

---
description: Enforce Node.js + Express + Prisma microservice folder structure
alwaysApply: true
---

# Backend Microservice Structure Rule

When generating a new backend service, enforce the following structure:

## 1️⃣ Root Structure

The service must follow this folder layout:

/
├── src/
│   ├── app.ts
│   ├── server.ts
│   │
│   ├── config/
│   │    ├── env.ts
│   │    └── prisma.ts
│   │
│   ├── modules/
│   │    └── <feature>/
│   │         ├── routes/
│   │         ├── controllers/
│   │         ├── services/
│   │         ├── repositories/ (optional)
│   │         ├── dto/
│   │         ├── validators/
│   │         └── index.ts
│   │
│   ├── middlewares/
│   │    ├── error.middleware.ts
│   │    └── auth.middleware.ts
│   │
│   ├── utils/
│   │
│   ├── types/
│   │
│   └── constants/
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│
├── tests/
│
├── .env
├── package.json
├── tsconfig.json
└── README.md


## 2️⃣ Architectural Rules

1. Follow layered pattern:
   routes → controllers → services → repositories → prisma

2. Controllers:
   - No business logic.
   - Only request validation + service invocation.

3. Services:
   - Contains business logic.
   - No direct access to req/res.
   - Calls repository layer.

4. Repositories:
   - Handles Prisma calls.
   - No business logic.

5. Modules must be feature-based and isolated.

6. Each module must export:
   - route
   - service
   - controller

7. Use dependency injection pattern where possible.

8. Use centralized error handling middleware.

9. All APIs must return standardized response format:
{
  success: boolean,
  data?: any,
  error?: string
}


## 3️⃣ Prisma Requirements

- schema.prisma must be inside prisma folder.
- Prisma client must be initialized in config/prisma.ts.
- Never instantiate Prisma inside controllers or services directly.
- Use singleton pattern for Prisma client.


## 4️⃣ Microservice Requirements

- Each microservice must:
  - Have independent prisma schema.
  - Have independent environment config.
  - Not depend on other service’s internal modules.
  - Communicate via API or message broker (not direct DB sharing).

- Shared logic must be extracted into shared library package (if needed).


## 5️⃣ Naming Conventions

- Files: kebab-case
- Classes: PascalCase
- Functions: camelCase
- Folders: lowercase


## 6️⃣ Production Readiness

- Include:
  - Health check endpoint
  - Logging system
  - Environment validation
  - Graceful shutdown handling

- Do not generate demo or temporary code.
- Do not include console.logs in production files.