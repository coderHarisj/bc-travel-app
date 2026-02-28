---
trigger: always_on
---

---
description: Ensure schema supports future extensibility
alwaysApply: true
---

# Database Evolution Strategy

1. Design schema using domain-based modular grouping.
2. Core tables must not depend on feature tables.
3. Feature tables may reference core tables.
4. Avoid breaking changes:
   - No destructive column removal.
   - Prefer soft deprecation.
5. Use version-controlled migrations.
6. All new feature tables must:
   - Follow naming conventions
   - Include audit fields
   - Respect indexing strategy
7. Design with future join capability in mind.
8. Do not overload existing tables for new feature logic.