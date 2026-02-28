---
description: Add new feature tables safely
---

1. Analyze feature PRD.
2. Identify:
   - New entities
   - Required relationships
3. Check:
   - Does this modify core tables?
   - If yes → propose safe alternative.
4. Generate new Prisma models.
5. Add relations without breaking existing schema.
6. Generate migration strategy.
7. Update ER diagram.
8. Provide impact analysis.