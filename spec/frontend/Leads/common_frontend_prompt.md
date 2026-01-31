# Common Frontend Technical Specification
## Tourism Management System

---

## 1. Purpose

This document defines the **foundational frontend architecture, security model, authorization strategy, coding standards, and scalability principles** for the Tourism Management System.

All page-level specifications MUST:
- Reference this document
- Reuse the same architectural patterns
- NOT redefine cross-cutting concerns such as:
  - Authentication
  - Authorization
  - Encryption
  - API standards
  - Folder structure

This document acts as the **single source of truth**.

---

## 2. Application Overview

- Application Type: Internal Operations Web Application
- Platform: Desktop-only SPA
- Users:
  - Admins
  - Internal Operations Team
- Core Modules:
  - Leads
  - Bookings
  - Packages
  - Settings

---

## 3. Technology Stack

- Next JS (JavaScript)
- Material UI (MUI)
- Redux Toolkit (Global State Management)
- Axios (API Communication)

---

## 4. Frontend Architecture

### 4.1 Architectural Pattern
- Feature-based modular architecture
- Clear separation of:
  - Layout
  - Pages
  - Components
  - State
  - Services
  - Security & Auth utilities

---

### 4.2 Mandatory Folder Structure

```text
src/
│
├── app/
│   ├── store/
│   ├── rootReducer.js
│   └── authSlice.js
│
├── layout/
│   ├── Header/
│   ├── Drawer/
│   └── AppLayout.jsx
│
├── features/
│   ├── leads/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── services/
│   │   ├── store/
│   │   └── utils/
│
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── constants/
│   └── utils/
│
├── security/
│   ├── encryption.js
│   ├── sanitization.js
│   └── auth.js
│
├── services/
│   ├── axiosInstance.js
│   └── tokenService.js
│
└── config/
    ├── env.js
    └── api.js

5. Coding Principles (Global Enforcement)

DRY: Shared utilities, hooks, reusable components

KISS: Simple, predictable UI behavior

SOLID (Frontend-Applicable):

SRP: One responsibility per component

OCP: Extend via composition/hooks

UI components must never contain:

API logic

Auth logic

Encryption logic

6. State Management (Redux)
6.1 Redux Usage Rules

Redux is used only for:

Auth & session state

Shared API data

Cross-page UI state

Page-level and form state remains local.

6.2 Auth Slice (JWT-Based)

authSlice manages:

Access token

Refresh token

Token expiry metadata

Authentication status

Tokens must:

Never be logged

Never be hardcoded

Never be passed via query params

7. API Communication & Authorization
7.1 Axios Standard

Single centralized Axios instance

No Axios usage inside UI components

Request & response interceptors mandatory

7.2 JWT Authorization Flow (Current Scope)

JWT access token received via response headers

Token attached to all API requests via Axios interceptor

Token passed only in request headers

7.3 Refresh Token Mechanism

Refresh token stored separately

Axios interceptor handles:

401 / token expiry

Token refresh

Retrying original request

Refresh failure triggers:

Auth state clear

Redirect to login (future)

8. Frontend Security Model
8.1 Encrypted API Communication

Encrypt request payloads before sending

Decrypt response payloads after receiving

Encryption handled inside Axios interceptors

Encryption logic isolated under /security

8.2 Application-Level Security

XSS prevention

Input sanitization

Safe rendering of user data

No sensitive data in logs or errors

8.3 PII Handling

Current:

No PII masking

Secure handling only

Future:

UI-level PII masking

9. Authentication UI (Future Scope)

Dedicated Login Page

Token acquisition & storage

Auth guards for protected routes

Login UI is out of current scope.

10. Testing Standards

Unit tests for:

Components

Redux slices

Axios interceptors

Encryption & auth utilities

11. Production Hardening

Secure logout

Token expiry handling

Session invalidation

No sensitive data in build artifacts