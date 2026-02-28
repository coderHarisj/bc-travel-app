---
trigger: always_on
---

# API Security Standards

## Mandatory Middleware
- Helmet
- Strict CORS
- Rate limiting
- Request ID injection

## Authentication
- Short-lived JWT access tokens
- Refresh tokens stored and validated via DB
- Token rotation enforced

## Authorization
- Explicit role checks
- No implicit trust of decoded tokens

## Forbidden
- Logging tokens
- Logging PII
- Trusting client-side claims
- Security by obscurity