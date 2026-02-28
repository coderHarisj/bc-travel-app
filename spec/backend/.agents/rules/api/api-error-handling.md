---
trigger: always_on
---

# API Error Handling Strategy

## Error Model
- All errors extend AppError
- Errors are typed and intentional
## Controller Rule
Controllers must not catch errors.
## Global Error Handler
Responsibilities:
- Map domain errors to HTTP status codes
- Mask internal details in production
- Preserve stack traces in non-prod
## Forbidden
- Throwing raw Error
- Leaking stack traces
- Inconsistent error response shapes

## Standard Error Response
{
  "error": {
    "code": "DOMAIN_ERROR_CODE",
    "message": "Human readable message",
    "requestId": "uuid"
  }
}