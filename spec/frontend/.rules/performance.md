# Performance Rules

## Must Implement
- Route-based code splitting
- React.memo for heavy components
- useMemo / useCallback where needed
- Debounce search (300ms)
- Lazy load heavy modules

## Tables
- Memoize rows
- Add virtualization for large datasets (future)

## Prohibited
- No lazy loading without route boundaries
- No inline event handlers in loops
- No unnecessary re-renders

