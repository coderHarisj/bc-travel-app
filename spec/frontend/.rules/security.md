# Security Rules

## Mandatory
- Sanitize all user input via sanitization.js
- No sensitive data in console logs
- No tokens stored in localStorage (if httpOnly used)

## Prevent
- XSS via dangerouslySetInnerHTML
- Hardcoded secrets
- Exposing env secrets in client bundle

## PII Handling
- Do not log user PII