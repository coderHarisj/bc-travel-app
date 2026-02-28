# Component Rules

## UI Components MUST NOT:
- Call APIs
- Access tokens
- Contain auth logic
- Directly dispatch Redux

## UI Components MUST:
- Be presentational
- Receive data via props
- Follow MUI theme
- Use proper PropTypes or strict typing

## Naming
- PascalCase for components
- Hooks prefixed with use