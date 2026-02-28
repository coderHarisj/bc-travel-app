# API Communication Rules

## All API calls:
- Must use axiosInstance
- Must use baseURL from env
- Must include JWT in Authorization header only

## Forbidden
- No tokens in query params
- No token logging
- No manual fetch usage

## Interceptor Enforcement
- Attach JWT automatically
- Handle 401 refresh flow
- Retry original request after refresh
- Clear auth state if refresh fails