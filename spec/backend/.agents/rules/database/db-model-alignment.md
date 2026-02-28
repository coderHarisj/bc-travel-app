---
trigger: always_on
---

---
description: Ensure data model aligns with business requirements
alwaysApply: true
---

# Data Model & Business Alignment Rules

1. Every entity must reflect a real business object.
2. Each field must map to a functional requirement.
3. Naming conventions:
   - Tables: snake_case plural (users, orders)
   - Columns: snake_case
4. Relationships must represent real-world logic.
5. Avoid unnecessary tables or unused fields.
6. Provide schema-level documentation comments.
7. Do not introduce speculative fields.