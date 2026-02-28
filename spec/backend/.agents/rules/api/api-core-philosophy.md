---
trigger: always_on
---

# API Core Philosophy (Non-Negotiable)

## Prime Law
Every API unit must be:
- Testable
- Replaceable
- Deletable
If removing an endpoint causes fear, side-effects, or cross-feature failures,
the architecture has already failed.
## Design Truths
- APIs are contracts, not implementations
- Stability > cleverness
- Explicit > implicit
- Predictable > flexible
## Immutable Rules
- No shared mutable state between requests
- No hidden side effects
- No behavior based on environment assumptions
## API Failure Standard
A failed request must:
- Fail fast
- Fail explicitly
- Fail with intent
Silent failures are forbidden.

# API Clean Code Constraints
## Function Rules
- Max 20 lines per function
- Max nesting depth: 2
- No magic numbers
- Names must explain intent
## File Rules
- One responsibility per file
- No god-objects
- No commented-out code
## Deletion Test
If deleting a feature:
- No unrelated tests break
- No other services require changes
If deletion hurts, design failed.

# Final API Law
APIs are contracts.
Contracts must be:
- Explicit
- Stable
- Predictable
If intent is unclear:
Refactor.
If behavior surprises:
Fix.
No exceptions.