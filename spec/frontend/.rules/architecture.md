# Architecture Rules - Tourism Management System

## Core Pattern
- Feature-based modular architecture
- Strict separation of concerns

## Folder Responsibilities
- layout/ → Application shell only
- features/ → Business logic grouped by domain
- shared/ → Reusable UI + hooks
- services/ → Axios + token handling only
- security/ → Sanitization + auth utilities
- config/ → Environment + endpoints

## Enforcement
- Pages orchestrate only
- No cross-feature imports
- Each feature must contain:
  - pages/
  - components/
  - services/
  - store/
  - utils/

## Prohibited
- No API logic inside UI components
- No direct token handling inside components
- No business logic inside layout/