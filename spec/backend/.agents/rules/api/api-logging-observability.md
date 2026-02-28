---
trigger: always_on
---

# API Logging & Observability

## Logging Rules
- Structured JSON logs only
- Every log includes requestId
- Log events, not noise
## Dynamic Configuration
- Logging behavior controlled via DB
- No redeployment required for changes
## What to Log
- Request received
- Business event occurred
- Operation failed
## Forbidden
- Logging passwords
- Logging full JWTs
- Logging raw request bodies