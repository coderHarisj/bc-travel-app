# Backend Technical Specification: Travel Management System - Leads Module

> **Version:** 1.0.0  
> **Status:** DRAFT  
> **Date:** 2026-01-31  
> **Audience:** Backend Engineers, DevOps, QA, Architecture Review Board  
> **Module:** Lead/Enquiry Management  
> **Expected Scale:** 10,000+ leads/month, 100+ concurrent users

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Context & Scope](#2-system-context--scope)
3. [Functional Requirements](#3-functional-requirements)
4. [Data Model & Schema](#4-data-model--schema)
5. [API Specification](#5-api-specification)
6. [Security Architecture](#6-security-architecture)
7. [Performance Requirements](#7-performance-requirements)
8. [Technical Architecture](#8-technical-architecture)
9. [Error Handling & Resilience](#9-error-handling--resilience)
10. [Implementation Roadmap](#10-implementation-roadmap)
11. [Deployment Strategy](#11-deployment-strategy)
12. [Compliance & Audit](#12-compliance--audit)
13. [Appendix: Code Examples](#13-appendix-code-examples)

---

## 1. Executive Summary

### Purpose

The **Leads Module** is the entry point of the travel sales pipeline. It captures incoming customer travel inquiries, organizes potential customers, and enables sales teams to convert leads into quotes and bookings.

### Key Capabilities

- **Lead Lifecycle Management:** Create, track, update, and convert leads through defined status flows
- **Intelligent Search & Filtering:** Full-text search, multi-criteria filtering, and date-range queries
- **Lead Conversion:** Seamless conversion from lead to quote/booking with data integrity
- **Role-Based Access:** Admin, Manager, Agent, and Viewer roles with granular permissions
- **Multi-Tenancy:** Isolated data per travel agency
- **Audit Trail:** Complete audit logging for compliance and troubleshooting
- **Scalability:** Support 500+ concurrent users, 5,000 leads/day creation rate

### Success Criteria

✅ All CRUD operations within SLA latency (<500ms for searches, <200ms for creates)  
✅ Zero unauthorized data access incidents  
✅ 99.5% uptime maintained  
✅ All PII encrypted at rest  
✅ Zero data loss during system failures  
✅ GDPR-compliant data handling  

---

## 2. System Context & Scope

### 2.1 Problem Statement

Travel agencies currently face:
- **Fragmented lead collection** (multiple channels, no central system)
- **Manual tracking overhead** (spreadsheets, lost emails)
- **Conversion bottlenecks** (unclear lead status, delayed follow-ups)
- **Duplicate leads** (same customer multiple inquiries)
- **No audit trail** (compliance issues, lost data integrity)

### 2.2 Solution Overview

The Leads Module provides a **centralized, API-first backend** that:

- Captures leads from multiple channels (web form, phone, email, API)
- Tracks lead progression through defined statuses
- Enables quick conversion to quotes and bookings
- Maintains complete audit trail for compliance
- Scales horizontally with stateless API design

### 2.3 MVP Scope (Phase 1: 1-2 weeks)

**In Scope:**

- Lead CRUD operations (Create, Read, Update, Delete with soft deletes)
- Search & filtering (name, phone, email, date range)
- Lead status lifecycle (NEW → CONTACTED → QUOTED → NEGOTIATING → BOOKED → REJECTED → ARCHIVED)
- Convert lead to quote (atomic transaction)
- Basic validations (email format, phone format, date logic)
- Cookie-based authentication
- Role-based access control (Admin, Agent, Manager, Viewer)
- Audit logging (who, what, when, old value, new value)

**Post-MVP (Phase 2-3):**

- Automated follow-up reminders
- Lead scoring algorithm
- CRM integrations (Salesforce, HubSpot)
- Bulk lead import/export (CSV)
- Activity timeline per lead
- Real-time notifications
- Advanced duplicate detection

### 2.4 Out of Scope

- ~~Advanced ML-based lead scoring~~ (Phase 3+)
- ~~WhatsApp/SMS integration~~ (Future phase)
- ~~Voice call recording~~ (Compliance review needed)
- ~~Third-party CRM sync~~ (Phase 2+)

---

## 3. Functional Requirements

### 3.1 Lead Management

#### 3.1.1 Create Lead

**Actors:** Admin, Manager, Agent  
**Preconditions:** User authenticated, has LEAD_CREATE permission  
**Workflow:**

1. User submits lead form with required fields
2. Backend validates all inputs
3. Backend checks for obvious duplicates (email + phone combination)
4. Creates new Lead record with DEFAULT status = "NEW"
5. Records created_by user ID and timestamp
6. Returns Lead object with generated ID

**Required Fields:**

| Field | Type | Validation | Example |
|-------|------|-----------|---------|
| `lead_name` | String | Required, max 100 chars, no special chars | "Mustaq Ahmed" |
| `email` | String | Required, valid email format, unique check with phone | "mustaq@example.com" |
| `phone` | String | Required, E.164 format, unique check with email | "+919876543210" |
| `country_code` | String | Required, ISO 3166-1 alpha-2 | "IN" |
| `travel_date` | ISO DateTime | Required, must be >= today | "2025-03-28T00:00:00Z" |
| `destination` | String | Required, foreign key to destinations | "Andaman" |
| `travel_type` | Enum | Required, one of: DOMESTIC, INTERNATIONAL | "DOMESTIC" |
| `num_travelers` | Integer | Required, range 1-500 | 10 |
| `budget` | Decimal | Optional, must be positive if provided | 150000.00 |
| `lead_source` | Enum | Required, one of: WEBSITE, PHONE, EMAIL, REFERRAL, SOCIAL, OTHER | "WEBSITE" |
| `notes` | String | Optional, max 1000 chars | "Family trip, budget flexible" |

**Response (201 Created):**

```json
{
  "success": true,
  "data": {
    "lead_id": "550e8400-e29b-41d4-a716-446655440000",
    "lead_name": "Mustaq Ahmed",
    "email": "mustaq@example.com",
    "phone": "+919876543210",
    "country_code": "IN",
    "travel_date": "2025-03-28",
    "destination": "Andaman",
    "travel_type": "DOMESTIC",
    "num_travelers": 10,
    "budget": 150000.00,
    "lead_source": "WEBSITE",
    "status": "NEW",
    "assigned_to": null,
    "created_by": "user-123",
    "created_at": "2026-01-31T10:30:00Z",
    "updated_at": "2026-01-31T10:30:00Z",
    "notes": "Family trip, budget flexible"
  }
}
```

**Error Scenarios:**

| Error | HTTP Code | Message |
|-------|-----------|---------|
| Validation failed | 400 | `{ "error": "Invalid email format", "field": "email" }` |
| Unauthorized | 403 | `{ "error": "LEAD_CREATE permission denied" }` |

#### 3.1.2 List Leads

**Actors:** Admin, Manager, Agent, Viewer  
**Access Rule:** Agents see only their assigned leads; Admin/Manager see all leads  
**Pagination:** Default 25 records, max 100 per page  

**Query Parameters:**

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `page` | Integer | Page number (1-based) | 1 |
| `limit` | Integer | Records per page (1-100) | 25 |
| `sort_by` | String | Field to sort: created_at, name, travel_date, status | created_at |
| `sort_order` | String | ASC or DESC | DESC |
| `search` | String | Free-text search (name, email, phone) | null |
| `status` | String | Filter by status | null |
| `travel_type` | String | Filter by DOMESTIC/INTERNATIONAL | null |
| `date_from` | ISO Date | Filter leads with travel_date >= | null |
| `date_to` | ISO Date | Filter leads with travel_date <= | null |
| `destination` | String | Filter by destination | null |
| `assigned_to` | UUID | Filter by assigned user | null |

**Example Request:**

```
GET /api/v1/leads?page=1&limit=10&status=NEW&date_from=2025-03-01&date_to=2025-03-31&sort_by=created_at&sort_order=DESC
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "leads": [
      {
        "lead_id": "550e8400-e29b-41d4-a716-446655440000",
        "lead_name": "Mustaq Ahmed",
        "email": "mustaq@example.com",
        "phone": "+919876543210",
        "destination": "Andaman",
        "travel_type": "DOMESTIC",
        "travel_date": "2025-03-28",
        "num_travelers": 10,
        "status": "NEW",
        "assigned_to": "user-456",
        "created_at": "2026-01-31T10:30:00Z",
        "budget": 150000.00
      },
      {
        "lead_id": "550e8400-e29b-41d4-a716-446655440001",
        "lead_name": "Salman Khan",
        "email": "salman@example.com",
        "phone": "+919876543211",
        "destination": "Goa",
        "travel_type": "DOMESTIC",
        "travel_date": "2025-04-15",
        "num_travelers": 5,
        "status": "CONTACTED",
        "assigned_to": "user-789",
        "created_at": "2026-01-30T14:20:00Z",
        "budget": 75000.00
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total_records": 245,
      "total_pages": 25,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

#### 3.1.3 Get Lead Details

**Actors:** All authenticated users (with appropriate permissions)  
**Access Rule:** Agents see only their assigned leads; Admin see all  

**Request:**

```
GET /api/v1/leads/{lead_id}
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "lead_id": "550e8400-e29b-41d4-a716-446655440000",
    "lead_name": "Mustaq Ahmed",
    "email": "mustaq@example.com",
    "phone": "+919876543210",
    "country_code": "IN",
    "travel_date": "2025-03-28",
    "duration_days": 7,
    "destination": "Andaman",
    "travel_type": "DOMESTIC",
    "num_travelers": 10,
    "budget": 150000.00,
    "lead_source": "WEBSITE",
    "status": "NEW",
    "assigned_to": {
      "user_id": "user-456",
      "name": "Arjun Singh",
      "email": "arjun@agency.com"
    },
    "created_by": {
      "user_id": "user-123",
      "name": "Admin",
      "email": "admin@agency.com"
    },
    "notes": "Family trip, budget flexible. Interested in water sports.",
    "last_contacted_at": "2026-01-31T09:00:00Z",
    "created_at": "2026-01-31T10:30:00Z",
    "updated_at": "2026-01-31T10:30:00Z",
    "deleted_at": null,
    "linked_quotes": ["quote-123", "quote-124"],
    "activity_log": [
      {
        "action": "CREATED",
        "by": "user-123",
        "at": "2026-01-31T10:30:00Z",
        "changes": null
      },
      {
        "action": "ASSIGNED",
        "by": "user-123",
        "at": "2026-01-31T10:35:00Z",
        "changes": { "assigned_to": [null, "user-456"] }
      }
    ]
  }
}
```

#### 3.1.4 Update Lead

**Actors:** Admin, Manager (own team), Agent (own leads)  
**Preconditions:** Lead must exist, not deleted, user has permission  
**Workflow:**

1. Validate all updateable fields (no status change via this endpoint)
2. Compare old vs new values
3. Create audit log entry
4. Invalidate cache
5. Return updated lead

**Updateable Fields:** lead_name, email, phone, travel_date, destination, num_travelers, budget, notes, assigned_to  
**Non-Updateable:** lead_id, created_at, created_by, deleted_at (use specific endpoints)

**Request:**

```json
PUT /api/v1/leads/{lead_id}

{
  "lead_name": "Mustaq Ahmed Updated",
  "budget": 200000.00,
  "notes": "Updated budget to 2L"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "lead_id": "550e8400-e29b-41d4-a716-446655440000",
    "lead_name": "Mustaq Ahmed Updated",
    "budget": 200000.00,
    "notes": "Updated budget to 2L",
    "updated_at": "2026-01-31T11:00:00Z",
    "changes": {
      "lead_name": {
        "old": "Mustaq Ahmed",
        "new": "Mustaq Ahmed Updated"
      },
      "budget": {
        "old": 150000.00,
        "new": 200000.00
      }
    }
  }
}
```

#### 3.1.5 Update Lead Status

**Actors:** Admin, Manager, Agent  
**Valid Transitions:**

```
NEW → CONTACTED, REJECTED
CONTACTED → QUOTED, REJECTED
QUOTED → NEGOTIATING, REJECTED
NEGOTIATING → BOOKED, REJECTED
BOOKED → CLOSED, ARCHIVED
REJECTED → null (terminal state)
ARCHIVED → null (terminal state)
CLOSED → null (terminal state)
```

**Request:**

```json
PATCH /api/v1/leads/{lead_id}/status

{
  "new_status": "CONTACTED",
  "reason": "Customer called to discuss itinerary options"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "data": {
    "lead_id": "550e8400-e29b-41d4-a716-446655440000",
    "old_status": "NEW",
    "new_status": "CONTACTED",
    "status_changed_at": "2026-01-31T11:05:00Z",
    "updated_by": "user-456"
  }
}
```

**Error Scenarios:**

| Error | HTTP Code |
|-------|-----------|
| Invalid transition | 400 |
| Lead not found | 404 |
| Permission denied | 403 |

#### 3.1.6 Delete Lead (Soft Delete)

**Actors:** Admin only  
**Preconditions:** Lead not already deleted  
**Workflow:**

1. Set deleted_at = current timestamp
2. Create audit log entry
3. Remove from search indexes
4. Return confirmation

**Request:**

```json
DELETE /api/v1/leads/{lead_id}

{
  "reason": "Duplicate entry - merged with lead-001"
}
```

**Response (200 OK):**

```json
{
  "success": true,
  "message": "Lead soft deleted successfully",
  "data": {
    "lead_id": "550e8400-e29b-41d4-a716-446655440000",
    "deleted_at": "2026-01-31T11:10:00Z",
    "deleted_by": "user-123",
    "reason": "Duplicate entry - merged with lead-001"
  }
}
```

#### 3.1.7 Convert Lead to Quote

**Actors:** Admin, Manager, Agent  
**Preconditions:**
- Lead exists and not deleted
- Lead status = "QUOTED" or "NEGOTIATING"
- Lead not already converted
- User has LEAD_CONVERT permission

**Workflow (Atomic Transaction):**

1. Validate lead is convertible
2. BEGIN TRANSACTION
3. Create Quote record (linked to lead)
4. Update Lead status to "CONVERTED"
5. Record conversion audit log
6. COMMIT TRANSACTION
7. Emit event for downstream (notifications, etc.)

**Request:**

```json
POST /api/v1/leads/{lead_id}/convert-to-quote

{
  "quote_note": "Customer approved itinerary and pricing",
  "estimated_quote_value": 200000.00
}
```

**Response (201 Created):**

```json
{
  "success": true,
  "message": "Lead successfully converted to quote",
  "data": {
    "lead_id": "550e8400-e29b-41d4-a716-446655440000",
    "quote_id": "quote-550e8400-e29b-41d4",
    "lead_status_updated": "CONVERTED",
    "quote_status": "PENDING_APPROVAL",
    "quote_value": 200000.00,
    "converted_at": "2026-01-31T11:15:00Z",
    "converted_by": "user-456"
  }
}
```

**Error Scenarios:**

| Error | HTTP Code |
|-------|-----------|
| Lead already converted | 409 |
| Invalid lead status for conversion | 400 |
| Lead not found | 404 |

### 3.2 Search & Filtering

#### 3.2.1 Full-Text Search

**Features:**

- Search across: lead_name, email, phone, destination
- Fuzzy matching for name (typo tolerance)
- Case-insensitive
- Pagination support
- Result relevance ranking

**Request:**

```
GET /api/v1/leads/search?q=mustaq&page=1&limit=10
```

**Response:**

```json
{
  "success": true,
  "data": {
    "query": "mustaq",
    "results": [
      {
        "lead_id": "550e8400-e29b-41d4-a716-446655440000",
        "lead_name": "Mustaq Ahmed",
        "email": "mustaq@example.com",
        "phone": "+919876543210",
        "relevance_score": 1.0,
        "matched_fields": ["lead_name"]
      }
    ],
    "total_results": 1,
    "search_time_ms": 45
  }
}
```

### 3.3 Lead Status Lifecycle

```
┌─────────────────────────────────────────────────────┐
│                 LEAD LIFECYCLE                      │
├─────────────────────────────────────────────────────┤
│                                                     │
│  NEW ──→ CONTACTED ──→ QUOTED ──→ NEGOTIATING      │
│   ↓          ↓           ↓           ↓             │
│   └──→ REJECTED (any step)           │             │
│                                      ↓             │
│                                   BOOKED ──→ CLOSED│
│                                      ↓             │
│                                   ARCHIVED         │
│                                                     │
└─────────────────────────────────────────────────────┘

Status Rules:
- NEW: Initial state when lead created
- CONTACTED: Agent has called/emailed customer
- QUOTED: Formal quote sent to customer
- NEGOTIATING: Back-and-forth with customer
- BOOKED: Confirmed booking, moved to booking system
- CLOSED: Completed travel, no further action
- REJECTED: Customer declined, no interest
- ARCHIVED: Stale/inactive leads (>90 days no contact)
```

---

## 4. Data Model & Schema

### 4.1 Entity-Relationship Diagram

```
┌──────────────────────┐        ┌──────────────────────┐
│      USER            │        │   ORGANIZATION       │
├──────────────────────┤        ├──────────────────────┤
│ user_id (PK)         │◄───────│ org_id (PK)          │
│ org_id (FK)          │        │ org_name             │
│ email                │        │ created_at           │
│ password_hash        │        └──────────────────────┘
│ role                 │
│ created_at           │
└──────────────────────┘
         │
         │ creates
         │ assigns to
         ▼
┌──────────────────────┐
│       LEAD           │
├──────────────────────┤
│ lead_id (PK)         │
│ org_id (FK)          │
│ lead_name            │
│ email                │
│ phone                │
│ country_code         │
│ travel_date          │
│ destination          │
│ travel_type          │
│ num_travelers        │
│ budget               │
│ lead_source          │
│ status               │
│ assigned_to (FK)     │
│ created_by (FK)      │
│ notes                │
│ created_at           │
│ updated_at           │
│ deleted_at           │
└──────────────────────┘
         │
         │ converts to
         ▼
┌──────────────────────┐
│       QUOTE          │
├──────────────────────┤
│ quote_id (PK)        │
│ lead_id (FK)         │
│ org_id (FK)          │
│ status               │
│ total_amount         │
│ created_at           │
│ updated_at           │
└──────────────────────┘

┌──────────────────────┐
│  AUDIT_LOG           │
├──────────────────────┤
│ audit_id (PK)        │
│ org_id (FK)          │
│ entity_type          │
│ entity_id (FK)       │
│ action               │
│ old_value            │
│ new_value            │
│ changed_by (FK)      │
│ changed_at           │
│ ip_address           │
│ user_agent           │
└──────────────────────┘
```

### 4.2 Lead Table Schema

```sql
CREATE TABLE leads (
    -- Identifiers
    lead_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id) ON DELETE CASCADE,
    
    -- Customer Information (PII - ENCRYPTED)
    lead_name VARCHAR(100) NOT NULL,
    email VARCHAR(254) NOT NULL,
    phone VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL,
    
    -- Travel Details
    travel_date DATE NOT NULL,
    duration_days INTEGER DEFAULT NULL,
    destination VARCHAR(100) NOT NULL,
    travel_type ENUM ('DOMESTIC', 'INTERNATIONAL') NOT NULL,
    num_travelers INTEGER NOT NULL CHECK (num_travelers >= 1 AND num_travelers <= 500),
    budget DECIMAL(12, 2) DEFAULT NULL CHECK (budget > 0 OR budget IS NULL),
    
    -- Lead Metadata
    lead_source ENUM ('WEBSITE', 'PHONE', 'EMAIL', 'REFERRAL', 'SOCIAL', 'OTHER') NOT NULL,
    status ENUM ('NEW', 'CONTACTED', 'QUOTED', 'NEGOTIATING', 'BOOKED', 'REJECTED', 'ARCHIVED', 'CLOSED') 
        NOT NULL DEFAULT 'NEW',
    notes TEXT DEFAULT NULL,
    
    -- Relationships
    assigned_to UUID DEFAULT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    created_by UUID NOT NULL REFERENCES users(user_id) ON DELETE RESTRICT,
    
    -- Tracking
    last_contacted_at TIMESTAMP DEFAULT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP DEFAULT NULL,
    
    -- Constraints
    UNIQUE (org_id, email, phone) DEFERRABLE INITIALLY DEFERRED,
    CHECK (travel_date >= CURRENT_DATE),
    CHECK (created_at <= updated_at)
);

-- Indexes for Performance
CREATE INDEX idx_leads_org_status ON leads(org_id, status) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_org_assigned ON leads(org_id, assigned_to) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_org_created ON leads(org_id, created_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_travel_date ON leads(travel_date) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_destination ON leads(destination) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_email_phone ON leads(email, phone) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_created_updated ON leads(created_at, updated_at);

-- Full-text search index
CREATE INDEX idx_leads_fts ON leads USING GIN (
    to_tsvector('english', 
        lead_name || ' ' || email || ' ' || phone || ' ' || destination
    )
);

-- Soft delete index
CREATE INDEX idx_leads_deleted ON leads(deleted_at) WHERE deleted_at IS NOT NULL;
```

### 4.3 Audit Log Schema

```sql
CREATE TABLE audit_logs (
    audit_id BIGSERIAL PRIMARY KEY,
    org_id UUID NOT NULL REFERENCES organizations(org_id) ON DELETE CASCADE,
    
    -- What was changed
    entity_type VARCHAR(50) NOT NULL, -- 'LEAD', 'QUOTE', 'USER'
    entity_id UUID NOT NULL, -- Foreign key value
    
    -- How it changed
    action VARCHAR(50) NOT NULL, -- 'CREATE', 'UPDATE', 'DELETE', 'STATUS_CHANGE'
    old_value JSONB DEFAULT NULL,
    new_value JSONB DEFAULT NULL,
    
    -- Who changed it
    changed_by UUID NOT NULL REFERENCES users(user_id),
    changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- Context
    ip_address INET DEFAULT NULL,
    user_agent VARCHAR(500) DEFAULT NULL,
    request_id UUID DEFAULT NULL,
    
    -- Retention
    retention_until TIMESTAMP NOT NULL DEFAULT (CURRENT_TIMESTAMP + INTERVAL '5 years')
);

CREATE INDEX idx_audit_org_entity ON audit_logs(org_id, entity_type, entity_id);
CREATE INDEX idx_audit_changed_at ON audit_logs(changed_at DESC);
CREATE INDEX idx_audit_changed_by ON audit_logs(changed_by);
CREATE INDEX idx_audit_retention ON audit_logs(retention_until) WHERE retention_until < CURRENT_TIMESTAMP;
```

### 4.4 Quote Table Schema (MVP Minimal)

```sql
CREATE TABLE quotes (
    quote_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    lead_id UUID NOT NULL UNIQUE REFERENCES leads(lead_id) ON DELETE RESTRICT,
    org_id UUID NOT NULL REFERENCES organizations(org_id) ON DELETE CASCADE,
    
    status ENUM ('PENDING_APPROVAL', 'SENT', 'ACCEPTED', 'REJECTED', 'EXPIRED') 
        NOT NULL DEFAULT 'PENDING_APPROVAL',
    total_amount DECIMAL(12, 2) DEFAULT NULL,
    
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL '7 days'),
    
    CHECK (total_amount > 0 OR total_amount IS NULL)
);

CREATE INDEX idx_quotes_lead_id ON quotes(lead_id);
CREATE INDEX idx_quotes_org_status ON quotes(org_id, status);
```

### 4.5 User & Organization Schema

```sql
CREATE TABLE organizations (
    org_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_name VARCHAR(200) NOT NULL,
    subscription_tier ENUM ('FREE', 'STARTER', 'PROFESSIONAL', 'ENTERPRISE') NOT NULL DEFAULT 'STARTER',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id UUID NOT NULL REFERENCES organizations(org_id) ON DELETE CASCADE,
    
    email VARCHAR(254) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    
    role ENUM ('ADMIN', 'MANAGER', 'AGENT', 'VIEWER') NOT NULL DEFAULT 'AGENT',
    
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMP DEFAULT NULL,
    
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE (org_id, email)
);

CREATE INDEX idx_users_org_role ON users(org_id, role) WHERE is_active = TRUE;
```

---

## 5. API Specification

### 5.1 Base URL & Versioning

```
Base URL: https://api.travelapp.com/api/v1
API Version: v1
Versioning Strategy: URI-based (/api/v1, /api/v2, etc.)
```

### 5.2 Authentication & Authorization

**Authentication Method:** Cookie-based sessions + CSRF tokens

**Required Headers:**

```
Authorization: Bearer <session_token>  (or via secure HTTP-only cookie)
Content-Type: application/json
X-CSRF-Token: <csrf_token>  (for POST/PUT/PATCH/DELETE)
```

**Response Headers:**

```
X-Request-ID: <uuid>  (for tracing)
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1643644800
```

### 5.3 Complete API Endpoints Reference

#### Leads Endpoints

| Method | Endpoint | Description | Auth | Rate Limit |
|--------|----------|-------------|------|-----------|
| GET | `/leads` | List leads with filters | Required | 100/min |
| POST | `/leads` | Create new lead | Required | 50/min |
| GET | `/leads/{lead_id}` | Get lead details | Required | 200/min |
| PUT | `/leads/{lead_id}` | Update lead | Required | 100/min |
| PATCH | `/leads/{lead_id}/status` | Update lead status | Required | 50/min |
| DELETE | `/leads/{lead_id}` | Soft delete lead | Admin | 20/min |
| POST | `/leads/{lead_id}/convert-to-quote` | Convert to quote | Required | 50/min |
| GET | `/leads/search` | Full-text search | Required | 200/min |

#### Analytics Endpoints (Phase 2)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/analytics/conversion-rate` | Conversion metrics |
| GET | `/analytics/lead-source-stats` | Source analysis |
| GET | `/analytics/response-time` | Team performance |

### 5.4 Request/Response Format

**All requests use JSON:**

```json
{
  "field_name": "value",
  "nested_object": {
    "property": "value"
  }
}
```

**All responses follow standard wrapper:**

```json
{
  "success": true,  // or false
  "data": { /* response data */ },
  "error": null,  // or { "code": "ERROR_CODE", "message": "Human readable message" }
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-01-31T11:30:00Z",
    "response_time_ms": 145
  }
}
```

---

## 6. Security Architecture

### 6.1 Authentication Flow

```
1. User submits email + password
2. Backend validates credentials (bcrypt comparison)
3. Backend creates session (signed JWT or server-side session)
4. Returns HTTP-only secure cookie
5. Frontend includes cookie in subsequent requests
6. Backend validates session on each request

Cookie Format (Secure HTTP-only):
- Name: session_id
- Value: <encrypted session token>
- Secure: true (HTTPS only)
- HttpOnly: true (JavaScript cannot access)
- SameSite: Strict (CSRF protection)
- Max-Age: 30 minutes
- Domain: .travelapp.com
```

### 6.2 Authorization Model

**Role-Based Access Control (RBAC):**

```
┌──────────┬─────────────┬──────────┬──────────┬────────┐
│ Resource │ ADMIN       │ MANAGER  │ AGENT    │ VIEWER │
├──────────┼─────────────┼──────────┼──────────┼────────┤
│ CREATE   │ ✅ (any)    │ ✅ own   │ ✅ own   │ ❌     │
│ READ     │ ✅ (all)    │ ✅ team  │ ✅ own   │ ✅ own │
│ UPDATE   │ ✅ (any)    │ ✅ team  │ ✅ own   │ ❌     │
│ DELETE   │ ✅          │ ❌       │ ❌       │ ❌     │
│ STATUS   │ ✅          │ ✅ team  │ ✅ own   │ ❌     │
│ CONVERT  │ ✅          │ ✅ team  │ ✅ own   │ ❌     │
│ SETTINGS │ ✅          │ ❌       │ ❌       │ ❌     │
└──────────┴─────────────┴──────────┴──────────┴────────┘
```

### 6.3 Data Protection

**PII Encryption:**

```
Fields to encrypt:
- email (indexed separately via hash)
- phone (indexed separately via hash)
- lead_name (optional, based on compliance)

Encryption Strategy:
- Algorithm: AES-256-GCM
- Key Management: AWS KMS (recommended) or HashiCorp Vault
- Rotation: Annual (with re-encryption)
- At-rest: Always encrypted
- In-transit: TLS 1.3+

Example Implementation (Node.js with crypto):

const crypto = require('crypto');

function encryptPII(plaintext, masterKey) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', Buffer.from(masterKey, 'hex'), iv);
  let encrypted = cipher.update(plaintext, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag();
  
  return {
    encrypted: encrypted,
    iv: iv.toString('hex'),
    authTag: authTag.toString('hex')
  };
}

function decryptPII(encryptedData, masterKey) {
  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    Buffer.from(masterKey, 'hex'),
    Buffer.from(encryptedData.iv, 'hex')
  );
  decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
  
  let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}
```

### 6.4 Input Validation & Sanitization

```javascript
// Input Validation Examples

const validateLeadInput = (input) => {
  const errors = {};
  
  // Email validation
  if (!input.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(input.email)) {
    errors.email = "Valid email required";
  }
  
  // Phone validation (E.164 format)
  if (!input.phone || !/^\+?[1-9]\d{1,14}$/.test(input.phone)) {
    errors.phone = "Valid E.164 phone number required";
  }
  
  // Travel date validation (not in past)
  if (!input.travel_date || new Date(input.travel_date) < new Date()) {
    errors.travel_date = "Travel date must be in future";
  }
  
  // Budget validation (if provided, must be positive)
  if (input.budget !== null && input.budget <= 0) {
    errors.budget = "Budget must be positive";
  }
  
  // Name validation (no special chars)
  if (!input.lead_name || !/^[a-zA-Z\s'-]+$/.test(input.lead_name)) {
    errors.lead_name = "Name contains invalid characters";
  }
  
  return Object.keys(errors).length > 0 ? errors : null;
};

// SQL Injection Prevention: Use Parameterized Queries

// UNSAFE (DO NOT USE)
const query = `SELECT * FROM leads WHERE email = '${userInput}'`;

// SAFE (Use prepared statements)
const query = `SELECT * FROM leads WHERE email = $1 AND deleted_at IS NULL`;
const result = await db.query(query, [userInput]);

// XSS Prevention: HTML Escape Output
const escapeHtml = (text) => {
  const map = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#039;'
  };
  return text.replace(/[&<>"']/g, (m) => map[m]);
};
```

### 6.5 GDPR Compliance

**Data Subject Rights:**

```
1. Right to Access
   - Endpoint: GET /api/v1/users/me/data
   - Returns: All personal data related to user
   - Response: JSON or CSV export
   - Timeline: Within 30 days

2. Right to Erasure ("Right to be Forgotten")
   - Endpoint: POST /api/v1/users/me/erasure-request
   - Action: Soft delete all PII, anonymize audit logs
   - Status: Tracked with erasure_request_id
   - Timeline: Within 30 days
   
   Implementation:
   - Flag user for erasure
   - Run anonymization job
   - Clear all cookies/sessions
   - Log erasure action with timestamp

3. Right to Rectification
   - Allow users to correct their data
   - Audit log captures all changes
   - Endpoint: PUT /api/v1/users/me

4. Data Portability
   - Endpoint: GET /api/v1/users/me/export
   - Format: JSON/CSV
   - Includes: All user leads, quotes, activity

5. Consent Management
   - Track consent timestamps
   - Maintain audit trail
   - Allow withdrawal
```

### 6.6 Audit Logging

**Complete Audit Trail:**

```sql
-- Example: Track every lead change
INSERT INTO audit_logs (
  org_id, entity_type, entity_id, action, 
  old_value, new_value, changed_by, changed_at,
  ip_address, user_agent, request_id
) VALUES (
  $1, 'LEAD', $2, 'UPDATE',
  jsonb_build_object('budget', 150000),
  jsonb_build_object('budget', 200000),
  $3, CURRENT_TIMESTAMP,
  $4, $5, $6
);

-- Query audit trail for lead
SELECT 
  audit_id, action, old_value, new_value, 
  changed_by, changed_at
FROM audit_logs
WHERE entity_type = 'LEAD' AND entity_id = $1
ORDER BY changed_at DESC;
```

---

## 7. Performance Requirements

### 7.1 Latency SLAs

| Operation | Target | Limit | Notes |
|-----------|--------|-------|-------|
| Create Lead | <200ms | <500ms | Form submission → DB commit |
| List Leads (page 1) | <500ms | <1000ms | With typical filters |
| Search (paginated) | <500ms | <1000ms | FTS query |
| Get Lead Details | <300ms | <700ms | With joined data |
| Update Lead | <200ms | <500ms | DB update + cache invalidation |
| Update Status | <150ms | <400ms | Simpler transaction |
| Convert to Quote | <300ms | <800ms | Multiple DB operations |
| Delete Lead (soft) | <150ms | <400ms | Mark + cache invalidation |

### 7.2 Scalability Targets

**Capacity Planning:**

```
Current Scale (MVP):
- 100 concurrent users
- 5,000 leads/day creation
- 500 leads/hour average
- 50 leads/hour per agent (average)
- Lead list avg size: 10,000-50,000 records

Future Scale (12 months):
- 500+ concurrent users
- 50,000 leads/day creation
- 2,000+ leads/hour
- Retention: 1 million+ leads

Infrastructure Scaling:
1. Horizontal API scaling: 2-4 backend instances behind load balancer
2. Database: PostgreSQL with read replicas for reporting
3. Cache: Redis cluster for session + query caching
4. Search: Elasticsearch for full-text search (Phase 2)
```

### 7.3 Caching Strategy

```
Layer 1: Application Level (Redis)
├─ Lead search results: 5 min TTL (cache key: leads:search:{hash(params)})
├─ Lead detail: 10 min TTL (cache key: leads:{lead_id})
├─ User sessions: 30 min TTL (cache key: session:{session_id})
├─ Destination list: 1 hour TTL (cache key: destinations)
└─ User permissions: 5 min TTL (cache key: user:perms:{user_id})

Layer 2: Database Level (Query Results)
├─ Prepared statements with query plan caching
├─ Connection pooling (20-50 connections)
└─ Index-driven queries

Layer 3: Browser Level
├─ Static assets: 1 year (Cache-Control: public, max-age=31536000)
├─ API responses: No-cache, must-revalidate (Cache-Control: private, max-age=0)

Cache Invalidation Strategy:
- On LEAD CREATE: Invalidate leads:search:* pattern
- On LEAD UPDATE: Invalidate leads:{lead_id}
- On LEAD DELETE: Invalidate leads:{lead_id} + leads:search:* pattern
- On LEAD STATUS CHANGE: Invalidate leads:{lead_id}
- Automatic TTL expiry as fallback
```

### 7.4 Database Optimization

**Index Strategy:**

```sql
-- Most critical indexes
CREATE INDEX CONCURRENTLY idx_leads_org_status 
  ON leads(org_id, status) WHERE deleted_at IS NULL;

CREATE INDEX CONCURRENTLY idx_leads_org_created 
  ON leads(org_id, created_at DESC) WHERE deleted_at IS NULL;

CREATE INDEX CONCURRENTLY idx_leads_assigned_user 
  ON leads(assigned_to, status) WHERE deleted_at IS NULL;

-- Full-text search index
CREATE INDEX CONCURRENTLY idx_leads_fts 
  ON leads USING GIN(to_tsvector('english', lead_name || ' ' || email));

-- Prevent N+1 queries: Always JOIN when needed
SELECT 
  l.*,
  u.name as assigned_name,
  creator.name as created_by_name
FROM leads l
LEFT JOIN users u ON l.assigned_to = u.user_id
LEFT JOIN users creator ON l.created_by = creator.user_id
WHERE l.org_id = $1 AND l.deleted_at IS NULL
LIMIT 25;
```

### 7.5 Load Testing Strategy

```bash
# Apache JMeter Test Plan
# Simulate 500 concurrent users creating and listing leads

# Thread Group: 500 users ramping up over 2 minutes
# Duration: 10 minutes

# Test Scenarios:
1. Create Lead: 20% of traffic
2. List Leads: 50% of traffic
3. Get Details: 20% of traffic
4. Update Lead: 10% of traffic

# Expected Results:
- P50 latency: <300ms
- P95 latency: <700ms
- P99 latency: <1500ms
- Error rate: <0.5%
- Throughput: >10,000 req/sec
```

---

## 8. Technical Architecture

### 8.1 Recommended Stack

**Backend Framework Options:**

```
Option 1 (Recommended): Node.js + Express/NestJS
├─ Pros: Fast prototyping, excellent async/await, npm ecosystem
├─ Cons: Single-threaded (mitigate with clustering)
├─ Best for: Rapid MVP, stateless API design
└─ Example: Express + Prisma ORM + PostgreSQL

Option 2: Python + FastAPI
├─ Pros: Developer velocity, rich libraries, async native
├─ Cons: Slightly slower than Go
├─ Best for: Data-heavy operations, complex validations
└─ Example: FastAPI + SQLAlchemy + PostgreSQL

Option 3: Go + Gin/Echo
├─ Pros: Best performance, easy deployment (single binary)
├─ Cons: Steeper learning curve
├─ Best for: High-throughput, minimal latency requirements
└─ Example: Gin + sqlc + PostgreSQL
```

**Recommended Stack for MVP:**

```
Backend: Node.js (v18+) + Express.js
├─ Runtime: Node.js v18+ LTS
├─ Framework: Express.js (4.x) or NestJS (9.x)
├─ ORM: Prisma (excellent DX, type-safe)
├─ Validation: Joi or Zod
├─ Auth: Passport.js with cookie-session
├─ Logging: Winston or Pino
└─ Testing: Jest + Supertest

Database:
├─ RDBMS: PostgreSQL 14+
├─ Connection Pool: node-postgres with pg pool
└─ Migrations: Flyway or db-migrate

Cache:
├─ Redis: 6.x or 7.x
└─ Client: ioredis or redis npm package

Message Queue (Phase 2):
├─ RabbitMQ or Bull (Redis-based)
└─ For: Async notifications, lead creation events

Search (Phase 2):
├─ Elasticsearch 8.x
└─ For: Full-text search optimization
```

### 8.2 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                   CLIENT TIER                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Web Browser │  │ Mobile App   │  │ Third-party  │  │
│  │  (React)     │  │ (React Native)  │ API Clients  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │ HTTPS/TLS 1.3
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   API GATEWAY TIER                      │
│  ┌──────────────────────────────────────────────────┐  │
│  │  API Gateway (Kong / nginx)                      │  │
│  │  - SSL/TLS termination                           │  │
│  │  - Rate limiting (100 req/min per user)          │  │
│  │  - CORS management                               │  │
│  │  - Request logging                               │  │
│  │  - IP whitelisting (optional)                    │  │
│  └──────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   API Pod 1  │  │   API Pod 2  │  │   API Pod 3  │  (Horizontally Scaled)
├──────────────┤  ├──────────────┤  ├──────────────┤
│  Express.js  │  │  Express.js  │  │  Express.js  │
│  + Prisma    │  │  + Prisma    │  │  + Prisma    │
│              │  │              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ Auth     │ │  │ │ Auth     │ │  │ │ Auth     │ │
│ │ Service  │ │  │ │ Service  │ │  │ │ Service  │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ Lead     │ │  │ │ Lead     │ │  │ │ Lead     │ │
│ │ Service  │ │  │ │ Service  │ │  │ │ Service  │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │
│ │ Quote    │ │  │ │ Quote    │ │  │ │ Quote    │ │
│ │ Service  │ │  │ │ Service  │ │  │ │ Service  │ │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │
└──────────────┘  └──────────────┘  └──────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │ Connection Pool
                           ▼
        ┌──────────────────────────────────────┐
        │       PostgreSQL Primary DB          │
        │  ┌────────────────────────────────┐  │
        │  │ leads                          │  │
        │  │ quotes                         │  │
        │  │ users                          │  │
        │  │ organizations                  │  │
        │  │ audit_logs                     │  │
        │  └────────────────────────────────┘  │
        └──────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Read Replica │  │ Read Replica │  │   Backup     │
│  (Analytics) │  │  (Analytics) │  │  (Daily)     │
└──────────────┘  └──────────────┘  └──────────────┘

        ┌──────────────────────────────────────┐
        │   Redis Cache Cluster                │
        │  ┌────────────────────────────────┐  │
        │  │ Sessions                       │  │
        │  │ Lead search results            │  │
        │  │ User permissions               │  │
        │  │ Rate limiting counters         │  │
        │  └────────────────────────────────┘  │
        └──────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│           MONITORING & LOGGING                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Prometheus   │  │ ELK Stack    │  │ APM Tool     │   │
│  │ (Metrics)    │  │ (Logs)       │  │ (Datadog)    │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
└──────────────────────────────────────────────────────────┘
```

### 8.3 API Service Architecture (Detailed)

```javascript
// Project Structure: Node.js + Express
├── src/
│   ├── config/
│   │   ├── database.js          (Prisma client)
│   │   ├── redis.js             (Redis client)
│   │   └── environment.js       (ENV vars validation)
│   │
│   ├── middleware/
│   │   ├── auth.js              (Session validation)
│   │   ├── authorization.js     (RBAC checks)
│   │   ├── errorHandler.js      (Global error catch)
│   │   ├── requestLogger.js     (Structured logging)
│   │   ├── rateLimit.js         (Rate limiting)
│   │   ├── cors.js              (CORS config)
│   │   └── csrf.js              (CSRF protection)
│   │
│   ├── routes/
│   │   ├── leads.routes.js      (Lead endpoints)
│   │   ├── quotes.routes.js     (Quote endpoints)
│   │   ├── auth.routes.js       (Auth endpoints)
│   │   └── index.js             (Route aggregation)
│   │
│   ├── controllers/
│   │   ├── leadsController.js   (Lead business logic)
│   │   ├── quotesController.js  (Quote business logic)
│   │   └── authController.js    (Auth logic)
│   │
│   ├── services/
│   │   ├── leadService.js       (DB access layer)
│   │   ├── quoteService.js      (Quote operations)
│   │   ├── authService.js       (Auth operations)
│   │   ├── cacheService.js      (Redis operations)
│   │   └── auditService.js      (Audit logging)
│   │
│   ├── validators/
│   │   ├── leadValidator.js     (Joi schemas)
│   │   ├── quoteValidator.js
│   │   └── authValidator.js
│   │
│   ├── utils/
│   │   ├── encryption.js        (PII encryption)
│   │   ├── errorCodes.js        (Error constants)
│   │   ├── logger.js            (Winston logger)
│   │   └── helpers.js           (General utilities)
│   │
│   └── app.js                   (Express app init)
│
├── prisma/
│   ├── schema.prisma            (Data model)
│   └── migrations/              (DB migrations)
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── .env.example
├── .env.production
├── package.json
├── docker-compose.yml
└── Dockerfile
```

---

## 9. Error Handling & Resilience

### 9.1 Error Response Format

**Standard Error Response:**

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input provided",
    "details": [
      {
        "field": "email",
        "message": "Email format is invalid"
      },
      {
        "field": "phone",
        "message": "Phone must be in E.164 format"
      }
    ]
  },
  "meta": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2026-01-31T11:30:00Z"
  }
}
```

### 9.2 Comprehensive Error Codes

| Code | HTTP | Description | Retry? | User Action |
|------|------|-------------|--------|-------------|
| `VALIDATION_ERROR` | 400 | Input validation failed | No | Fix input and retry |
| `UNAUTHORIZED` | 401 | Session expired | Yes | Re-login |
| `PERMISSION_DENIED` | 403 | Insufficient permissions | No | Contact admin |
| `NOT_FOUND` | 404 | Resource not found | No | Verify ID |
| `CONFLICT` | 409 | Business logic conflict | No | See details |
| `RATE_LIMITED` | 429 | Too many requests | Yes | Wait and retry |
| `INTERNAL_ERROR` | 500 | Unexpected server error | Yes | Try again later |
| `SERVICE_UNAVAILABLE` | 503 | Service down | Yes | Retry after delay |
| `GATEWAY_TIMEOUT` | 504 | Request timeout | Yes | Retry with backoff |

### 9.3 Resilience Patterns

**Exponential Backoff with Jitter:**

```javascript
async function retryWithBackoff(fn, maxAttempts = 3) {
  let attempt = 0;
  const delays = [1000, 2000, 4000, 8000]; // 1s, 2s, 4s, 8s
  
  while (attempt < maxAttempts) {
    try {
      return await fn();
    } catch (error) {
      attempt++;
      
      if (attempt >= maxAttempts) throw error;
      if (!isRetryableError(error)) throw error;
      
      const delay = delays[attempt - 1];
      const jitter = Math.random() * 0.1 * delay; // ±10% jitter
      await sleep(delay + jitter);
    }
  }
}

// Usage
await retryWithBackoff(async () => {
  return await database.query('SELECT * FROM leads LIMIT 1');
}, 3);
```

**Circuit Breaker Pattern:**

```javascript
class CircuitBreaker {
  constructor(fn, options = {}) {
    this.fn = fn;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.threshold = options.threshold || 5;
    this.timeout = options.timeout || 60000; // 60s
    this.lastFailureTime = null;
  }
  
  async execute(...args) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = 'HALF_OPEN';
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }
    
    try {
      const result = await this.fn(...args);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failures++;
    this.lastFailureTime = Date.now();
    
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
    }
  }
}
```

### 9.4 Graceful Degradation

```javascript
// Lead Service with fallback caching
async function getLeadWithFallback(leadId) {
  try {
    // Try primary database first
    return await database.leads.findUnique({ where: { lead_id: leadId } });
  } catch (dbError) {
    logger.error('Database error, checking cache', { leadId, error: dbError });
    
    // Fallback: Check Redis cache
    const cachedLead = await redis.get(`leads:${leadId}`);
    if (cachedLead) {
      logger.warn('Serving stale lead from cache', { leadId });
      return JSON.parse(cachedLead);
    }
    
    // No fallback available, return error
    throw new ServiceUnavailableError('Database and cache unavailable');
  }
}
```

### 9.5 Monitoring & Alerting

**Prometheus Metrics:**

```javascript
const leadCreateDuration = new prometheus.Histogram({
  name: 'lead_create_duration_ms',
  help: 'Duration of lead creation in milliseconds',
  buckets: [50, 100, 200, 500, 1000],
  labelNames: ['status']
});

const leadListErrors = new prometheus.Counter({
  name: 'lead_list_errors_total',
  help: 'Total count of lead list errors',
  labelNames: ['error_type']
});

// Usage
const startTime = Date.now();
try {
  const lead = await createLead(leadData);
  leadCreateDuration.labels('success').observe(Date.now() - startTime);
} catch (error) {
  leadCreateDuration.labels('error').observe(Date.now() - startTime);
  leadListErrors.labels(error.code).inc();
}
```

**Alerting Rules:**

```yaml
groups:
  - name: lead_service
    rules:
      - alert: HighErrorRate
        expr: rate(lead_list_errors_total[5m]) > 0.05
        for: 5m
        annotations:
          summary: "Lead service error rate >5%"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, lead_create_duration_ms_bucket) > 500
        for: 5m
        annotations:
          summary: "Lead creation p95 latency >500ms"
      
      - alert: RateLimitExceeded
        expr: increase(rate_limit_exceeded_total[1m]) > 10
        for: 1m
        annotations:
          summary: "Rate limiting triggered >10 times/min"
```

---

## 10. Implementation Roadmap

### Phase 1: MVP Core (1-2 weeks)

**Scope:**
- ✅ Lead CRUD APIs (Create, Read, Update, Delete)
- ✅ Lead listing with basic search & filters
- ✅ Soft delete functionality
- ✅ Lead status lifecycle (NEW → CONTACTED → CONVERTED)
- ✅ Convert lead to quote (atomic transaction)
- ✅ Cookie-based authentication
- ✅ Basic role-based access control (Admin, Agent, Manager, Viewer)
- ✅ Input validation & error handling
- ✅ Audit logging

**Deliverables:**
- API endpoints fully functional
- Database schema & migrations
- Basic test coverage (70%+)
- API documentation (Swagger)
- Deployment guide

**Timeline:**
```
Week 1:
- Day 1-2: Database schema, ERD design
- Day 2-3: Auth & middleware setup
- Day 3-4: Lead CRUD endpoints
- Day 4-5: Testing & documentation

Week 2:
- Day 1: Lead search & filtering
- Day 2: Quote conversion flow
- Day 3: Audit logging
- Day 4: Load testing
- Day 5: Final deployment
```

### Phase 2: Stability & Scaling (1 week, concurrent with Phase 1)

**Scope:**
- ✅ Database index optimization
- ✅ Redis caching strategy
- ✅ Connection pooling
- ✅ Rate limiting
- ✅ Better error messages
- ✅ Monitoring & logging setup
- ✅ Better duplicate detection
- ✅ Status transition validation

**Deliverables:**
- Performance benchmarks (p50, p95, p99 latency)
- Load testing results
- Monitoring dashboard
- Caching strategy documentation

### Phase 3: Advanced Features (Post-MVP)

**Scope (Roadmap):**
- 🔮 Automated follow-up reminders
- 🔮 Lead scoring algorithm
- 🔮 CRM integrations (Salesforce, HubSpot)
- 🔮 Bulk lead import/export (CSV)
- 🔮 Activity timeline per lead
- 🔮 Real-time notifications (WebSocket)
- 🔮 Advanced analytics & reporting
- 🔮 Lead reassignment workflows
- 🔮 Integration tests with external APIs
- 🔮 A/B testing framework

---

## 11. Deployment Strategy

### 11.1 Container & Orchestration

**Dockerfile:**

```dockerfile
# Multi-stage build for optimized image size

FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app

# Install security updates
RUN apk add --no-cache dumb-init tini

# Non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Copy from builder
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"

USER nodejs

# Use dumb-init to handle signals properly
ENTRYPOINT ["/sbin/dumb-init", "--"]
CMD ["node", "src/app.js"]
```

**Docker Compose (Development):**

```yaml
version: '3.9'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://user:pass@db:5432/leads_db
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - .:/app
      - /app/node_modules
    command: npm run dev

  db:
    image: postgres:14-alpine
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=leads_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # pgAdmin for database inspection
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - db

volumes:
  postgres_data:
```

### 11.2 Kubernetes Deployment

**k8s Manifests:**

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: leads-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: leads-api
  template:
    metadata:
      labels:
        app: leads-api
    spec:
      containers:
      - name: api
        image: gcr.io/project/leads-api:1.0.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: redis-url
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: leads-api-service
spec:
  selector:
    app: leads-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: leads-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: leads-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 11.3 Zero-Downtime Deployment

**Blue-Green Deployment Strategy:**

```
1. Blue Environment (current, serving traffic)
   - Version: 1.5.0
   - 3 replicas
   - Handling 100% of traffic

2. Green Environment (new, warm-up)
   - Version: 1.6.0
   - 3 replicas
   - Database migrations run
   - Smoke tests verify
   - 0% of traffic initially

3. Switching (after validation)
   - Smoke tests pass
   - Health checks pass
   - Switch load balancer to Green: 10% traffic
   - Monitor for errors (5 min)
   - If OK: 50% traffic
   - If OK: 100% traffic
   - Keep Blue as rollback for 1 hour

4. Rollback (if issues)
   - Switch back to Blue immediately
   - Alert team
   - Investigate issue in Green environment
```

### 11.4 Database Migrations

**Flyway vs db-migrate vs Prisma Migrations:**

```
Recommendation: Prisma Migrations (for MVP)

Why:
- Automatic rollback capability
- Migration versioning built-in
- Integrated with Prisma schema
- Easy to generate from schema changes

Command:
npx prisma migrate dev --name add_lead_table
npx prisma migrate deploy  # production

Prisma Migration Structure:
migrations/
├── 20260131105000_create_leads_table/
│   └── migration.sql
├── 20260131110000_add_audit_logs/
│   └── migration.sql
└── migration_lock.toml
```

---

## 12. Compliance & Audit

### 12.1 Audit Logging Strategy

**Complete Audit Trail Requirements:**

```sql
-- Audit table design
CREATE TABLE audit_logs (
  audit_id BIGSERIAL PRIMARY KEY,
  org_id UUID NOT NULL,
  entity_type VARCHAR(50),      -- LEAD, QUOTE, USER
  entity_id UUID,
  action VARCHAR(50),            -- CREATE, UPDATE, DELETE, STATUS_CHANGE
  old_value JSONB,
  new_value JSONB,
  changed_by UUID NOT NULL,
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ip_address INET,
  user_agent VARCHAR(500),
  request_id UUID,
  retention_until TIMESTAMP
);

-- Example audit entries
-- User creates a lead
INSERT INTO audit_logs VALUES (
  DEFAULT, 'org-123', 'LEAD', 'lead-001', 'CREATE',
  NULL,
  '{"name":"Mustaq Ahmed","email":"...","phone":"..."}',
  'user-456',
  CURRENT_TIMESTAMP,
  '192.168.1.1',
  'Mozilla/5.0...',
  'req-uuid-123',
  CURRENT_TIMESTAMP + INTERVAL '5 years'
);

-- User updates lead status
INSERT INTO audit_logs VALUES (
  DEFAULT, 'org-123', 'LEAD', 'lead-001', 'STATUS_CHANGE',
  '{"status":"NEW"}',
  '{"status":"CONTACTED"}',
  'user-789',
  CURRENT_TIMESTAMP,
  '192.168.1.2',
  'Mozilla/5.0...',
  'req-uuid-124',
  CURRENT_TIMESTAMP + INTERVAL '5 years'
);
```

### 12.2 Data Retention & Deletion

**Retention Policy:**

```
Active Leads: Kept indefinitely (business requirement)
Archived Leads: Kept for 2 years, then deleted
Deleted Leads: Anonymized immediately, audit trail kept for 5 years
User Data: User can request deletion anytime (GDPR)
Audit Logs: Kept for 5 years (compliance)
Session Data: Deleted after 30 minutes of inactivity
```

### 12.3 GDPR Data Subject Rights Implementation

**Right to Erasure Flow:**

```
1. User requests data deletion
   POST /api/v1/users/me/erasure-request
   Response: { erasure_request_id: "uuid", status: "PENDING" }

2. Backend workflow:
   - Create erasure_requests record
   - Send confirmation email (7-day grace period)
   - If confirmed:
     a. Anonymize all user-created leads
     b. Remove user from assignment records
     c. Delete user account
     d. Clear all sessions
     e. Log erasure with timestamp
   - Status update to "COMPLETED"

3. Data anonymization process:
   - Replace lead names: "User #12345"
   - Replace emails: "deleted+12345@example.com"
   - Replace phone: "XXXXXX LAST4"
   - Keep travel details (aggregate analytics)
   - Preserve audit trail (for compliance)
```

---

## 13. Appendix: Code Examples

### 13.1 Express.js Controller Example

```javascript
// leadsController.js
const { validationResult } = require('express-validator');
const LeadService = require('../services/leadService');
const AuditService = require('../services/auditService');
const logger = require('../utils/logger');

class LeadsController {
  async createLead(req, res, next) {
    try {
      // Validate input
      const errors = validationResult(req);
      if (!errors.isEmpty()) {
        return res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            details: errors.array()
          }
        });
      }

      const { lead_name, email, phone, travel_date, destination, ...rest } = req.body;
      const userId = req.user.id;
      const orgId = req.user.org_id;

      // Check for duplicate
      const existing = await LeadService.findDuplicate(orgId, email, phone);
      if (existing) {
        return res.status(409).json({
          success: false,
          error: {
            code: 'DUPLICATE_LEAD',
            message: 'Lead with this email/phone already exists'
          }
        });
      }

      // Create lead
      const lead = await LeadService.createLead({
        org_id: orgId,
        lead_name,
        email,
        phone,
        travel_date,
        destination,
        created_by: userId,
        ...rest
      });

      // Log audit
      await AuditService.logAction({
        org_id: orgId,
        entity_type: 'LEAD',
        entity_id: lead.lead_id,
        action: 'CREATE',
        new_value: lead,
        changed_by: userId,
        ip_address: req.ip,
        user_agent: req.get('user-agent'),
        request_id: req.id
      });

      logger.info('Lead created', {
        lead_id: lead.lead_id,
        org_id: orgId,
        user_id: userId
      });

      res.status(201).json({
        success: true,
        data: lead
      });
    } catch (error) {
      next(error);
    }
  }

  async listLeads(req, res, next) {
    try {
      const { page = 1, limit = 25, status, search, sort_by = 'created_at' } = req.query;
      const userId = req.user.id;
      const orgId = req.user.org_id;
      const userRole = req.user.role;

      // Build filter
      const filters = {
        org_id: orgId,
        deleted_at: null
      };

      if (status) filters.status = status;
      if (userRole === 'AGENT') filters.assigned_to = userId; // Agents see only own leads

      // Get paginated results
      const { leads, total } = await LeadService.listLeads({
        filters,
        search,
        page: parseInt(page),
        limit: parseInt(limit),
        sort_by,
        sort_order: 'DESC'
      });

      res.json({
        success: true,
        data: {
          leads,
          pagination: {
            page: parseInt(page),
            limit: parseInt(limit),
            total_records: total,
            total_pages: Math.ceil(total / limit),
            has_next: page * limit < total,
            has_prev: page > 1
          }
        }
      });
    } catch (error) {
      next(error);
    }
  }

  async convertToQuote(req, res, next) {
    const client = await pool.connect();
    try {
      const { lead_id } = req.params;
      const { quote_note, estimated_quote_value } = req.body;
      const userId = req.user.id;
      const orgId = req.user.org_id;

      await client.query('BEGIN');

      // Get lead
      const lead = await client.query(
        'SELECT * FROM leads WHERE lead_id = $1 AND org_id = $2 AND deleted_at IS NULL',
        [lead_id, orgId]
      );

      if (lead.rows.length === 0) {
        await client.query('ROLLBACK');
        return res.status(404).json({
          success: false,
          error: { code: 'NOT_FOUND', message: 'Lead not found' }
        });
      }

      if (lead.rows[0].status === 'CONVERTED') {
        await client.query('ROLLBACK');
        return res.status(409).json({
          success: false,
          error: { code: 'ALREADY_CONVERTED', message: 'Lead already converted' }
        });
      }

      // Create quote
      const quote = await client.query(
        `INSERT INTO quotes (lead_id, org_id, total_amount) 
         VALUES ($1, $2, $3) RETURNING *`,
        [lead_id, orgId, estimated_quote_value]
      );

      // Update lead status
      await client.query(
        `UPDATE leads SET status = 'CONVERTED', updated_at = NOW() WHERE lead_id = $1`,
        [lead_id]
      );

      // Audit log
      await client.query(
        `INSERT INTO audit_logs (org_id, entity_type, entity_id, action, old_value, new_value, changed_by)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        [
          orgId,
          'LEAD',
          lead_id,
          'CONVERT_TO_QUOTE',
          JSON.stringify({ status: lead.rows[0].status }),
          JSON.stringify({ status: 'CONVERTED', quote_id: quote.rows[0].quote_id }),
          userId
        ]
      );

      await client.query('COMMIT');

      res.status(201).json({
        success: true,
        data: {
          lead_id,
          quote_id: quote.rows[0].quote_id,
          converted_at: new Date().toISOString()
        }
      });
    } catch (error) {
      await client.query('ROLLBACK');
      next(error);
    } finally {
      client.release();
    }
  }
}

module.exports = new LeadsController();
```

### 13.2 Prisma Schema Example

```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Organization {
  org_id          String   @id @default(uuid())
  org_name        String
  subscription_tier String @default("STARTER")
  created_at      DateTime @default(now())
  updated_at      DateTime @updatedAt

  users           User[]
  leads           Lead[]
  quotes          Quote[]
  audit_logs      AuditLog[]

  @@map("organizations")
}

model User {
  user_id         String   @id @default(uuid())
  org_id          String
  org             Organization @relation(fields: [org_id], references: [org_id], onDelete: Cascade)
  
  email           String
  password_hash   String
  name            String
  role            UserRole @default(AGENT)
  is_active       Boolean  @default(true)
  last_login_at   DateTime?
  
  created_at      DateTime @default(now())
  updated_at      DateTime @updatedAt
  
  leads_created   Lead[]   @relation("created_by")
  leads_assigned  Lead[]   @relation("assigned_to")
  audit_logs      AuditLog[]

  @@unique([org_id, email])
  @@map("users")
}

model Lead {
  lead_id         String   @id @default(uuid())
  org_id          String
  org             Organization @relation(fields: [org_id], references: [org_id], onDelete: Cascade)
  
  lead_name       String
  email           String
  phone           String
  country_code    String
  
  travel_date     DateTime
  duration_days   Int?
  destination     String
  travel_type     TravelType
  num_travelers   Int
  budget          Decimal?
  
  lead_source     LeadSource
  status          LeadStatus  @default(NEW)
  notes           String?
  
  assigned_to     String?
  assigned_user   User?    @relation("assigned_to", fields: [assigned_to], references: [user_id], onDelete: SetNull)
  created_by      String
  creator         User     @relation("created_by", fields: [created_by], references: [user_id], onDelete: Restrict)
  
  last_contacted_at DateTime?
  created_at      DateTime @default(now())
  updated_at      DateTime @updatedAt
  deleted_at      DateTime?
  
  quotes          Quote[]
  audit_logs      AuditLog[]

  @@unique([org_id, email, phone])
  @@index([org_id, status])
  @@index([org_id, created_at(sort: Desc)])
  @@index([assigned_to])
  @@index([travel_date])
  @@map("leads")
}

model Quote {
  quote_id        String   @id @default(uuid())
  lead_id         String   @unique
  lead            Lead     @relation(fields: [lead_id], references: [lead_id], onDelete: Restrict)
  org_id          String
  org             Organization @relation(fields: [org_id], references: [org_id], onDelete: Cascade)
  
  status          QuoteStatus @default(PENDING_APPROVAL)
  total_amount    Decimal?
  
  created_at      DateTime @default(now())
  updated_at      DateTime @updatedAt
  expires_at      DateTime @default(dbgenerated("CURRENT_TIMESTAMP + INTERVAL '7 days'"))

  @@index([org_id, status])
  @@map("quotes")
}

model AuditLog {
  audit_id        BigInt   @id @default(autoincrement())
  org_id          String
  org             Organization @relation(fields: [org_id], references: [org_id], onDelete: Cascade)
  
  entity_type     String   // LEAD, QUOTE, USER
  entity_id       String
  
  action          String   // CREATE, UPDATE, DELETE, STATUS_CHANGE
  old_value       Json?
  new_value       Json?
  
  changed_by      String
  changed_user    User     @relation(fields: [changed_by], references: [user_id], onDelete: Restrict)
  changed_at      DateTime @default(now())
  
  ip_address      String?
  user_agent      String?
  request_id      String?
  
  retention_until DateTime @default(dbgenerated("CURRENT_TIMESTAMP + INTERVAL '5 years'"))

  @@index([org_id, entity_type, entity_id])
  @@index([changed_at(sort: Desc)])
  @@map("audit_logs")
}

enum UserRole {
  ADMIN
  MANAGER
  AGENT
  VIEWER
}

enum LeadStatus {
  NEW
  CONTACTED
  QUOTED
  NEGOTIATING
  BOOKED
  REJECTED
  ARCHIVED
  CLOSED
}

enum QuoteStatus {
  PENDING_APPROVAL
  SENT
  ACCEPTED
  REJECTED
  EXPIRED
}

enum TravelType {
  DOMESTIC
  INTERNATIONAL
}

enum LeadSource {
  WEBSITE
  PHONE
  EMAIL
  REFERRAL
  SOCIAL
  OTHER
}
```

### 13.3 Integration Test Example

```javascript
// tests/integration/leads.test.js
const request = require('supertest');
const app = require('../../src/app');
const pool = require('../../src/config/database');

describe('Leads API Integration Tests', () => {
  let agent;
  let authToken;
  let testOrgId;
  let testUserId;

  beforeAll(async () => {
    // Setup: Create test org and user
    const orgResult = await pool.query(
      `INSERT INTO organizations (org_name) VALUES ('Test Agency') RETURNING org_id`
    );
    testOrgId = orgResult.rows[0].org_id;

    const userResult = await pool.query(
      `INSERT INTO users (org_id, email, password_hash, name, role) 
       VALUES ($1, $2, $3, $4, $5) RETURNING user_id`,
      [testOrgId, 'test@agency.com', 'hashed_password', 'Test User', 'AGENT']
    );
    testUserId = userResult.rows[0].user_id;

    // Create session
    agent = request.agent(app);
    authToken = generateSessionToken(testUserId, testOrgId);
  });

  afterAll(async () => {
    // Cleanup
    await pool.query(`DELETE FROM leads WHERE org_id = $1`, [testOrgId]);
    await pool.query(`DELETE FROM users WHERE user_id = $1`, [testUserId]);
    await pool.query(`DELETE FROM organizations WHERE org_id = $1`, [testOrgId]);
    await pool.end();
  });

  describe('POST /api/v1/leads', () => {
    it('should create a lead successfully', async () => {
      const response = await agent
        .post('/api/v1/leads')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          lead_name: 'Mustaq Ahmed',
          email: 'mustaq@example.com',
          phone: '+919876543210',
          country_code: 'IN',
          travel_date: '2025-03-28',
          destination: 'Andaman',
          travel_type: 'DOMESTIC',
          num_travelers: 10,
          budget: 150000.00,
          lead_source: 'WEBSITE'
        });

      expect(response.status).toBe(201);
      expect(response.body.success).toBe(true);
      expect(response.body.data.lead_id).toBeDefined();
      expect(response.body.data.status).toBe('NEW');
    });

    it('should reject invalid email', async () => {
      const response = await agent
        .post('/api/v1/leads')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          lead_name: 'Test User',
          email: 'invalid-email',
          phone: '+919876543210',
          country_code: 'IN',
          travel_date: '2025-03-28',
          destination: 'Andaman',
          travel_type: 'DOMESTIC',
          num_travelers: 5,
          budget: 100000.00,
          lead_source: 'WEBSITE'
        });

      expect(response.status).toBe(400);
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });

    it('should reject past travel date', async () => {
      const pastDate = new Date();
      pastDate.setDate(pastDate.getDate() - 1);

      const response = await agent
        .post('/api/v1/leads')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          lead_name: 'Test User',
          email: 'test@example.com',
          phone: '+919876543210',
          country_code: 'IN',
          travel_date: pastDate.toISOString().split('T')[0],
          destination: 'Andaman',
          travel_type: 'DOMESTIC',
          num_travelers: 5,
          budget: 100000.00,
          lead_source: 'WEBSITE'
        });

      expect(response.status).toBe(400);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });
  });

  describe('GET /api/v1/leads', () => {
    beforeEach(async () => {
      // Create test leads
      await pool.query(
        `INSERT INTO leads (org_id, lead_name, email, phone, country_code, travel_date, 
         destination, travel_type, num_travelers, lead_source, created_by)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)`,
        [testOrgId, 'Lead 1', 'lead1@test.com', '+919876543211', 'IN', '2025-04-01',
         'Goa', 'DOMESTIC', 5, 'WEBSITE', testUserId]
      );
    });

    it('should list leads for user org', async () => {
      const response = await agent
        .get('/api/v1/leads?page=1&limit=10')
        .set('Authorization', `Bearer ${authToken}`);

      expect(response.status).toBe(200);
      expect(response.body.success).toBe(true);
      expect(response.body.data.leads).toBeInstanceOf(Array);
      expect(response.body.data.pagination.total_records).toBeGreaterThan(0);
    });

    it('should filter by status', async () => {
      const response = await agent
        .get('/api/v1/leads?status=NEW')
        .set('Authorization', `Bearer ${authToken}`);

      expect(response.status).toBe(200);
      expect(response.body.data.leads).toEqual(
        expect.arrayContaining([
          expect.objectContaining({ status: 'NEW' })
        ])
      );
    });
  });

  describe('POST /api/v1/leads/{lead_id}/convert-to-quote', () => {
    it('should convert lead to quote', async () => {
      // Create a lead
      const leadResult = await pool.query(
        `INSERT INTO leads (org_id, lead_name, email, phone, country_code, travel_date, 
         destination, travel_type, num_travelers, status, lead_source, created_by)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12) RETURNING lead_id`,
        [testOrgId, 'Test Lead', 'convert@test.com', '+919876543212', 'IN', '2025-05-01',
         'Kerala', 'DOMESTIC', 3, 'QUOTED', 'WEBSITE', testUserId]
      );
      const leadId = leadResult.rows[0].lead_id;

      const response = await agent
        .post(`/api/v1/leads/${leadId}/convert-to-quote`)
        .set('Authorization', `Bearer ${authToken}`)
        .send({ estimated_quote_value: 200000.00 });

      expect(response.status).toBe(201);
      expect(response.body.success).toBe(true);
      expect(response.body.data.quote_id).toBeDefined();
    });
  });
});
```

---

## Summary

This comprehensive backend technical specification provides:

✅ **Complete functional requirements** with detailed API endpoints  
✅ **Production-grade data model** with security considerations  
✅ **Robust error handling** and resilience patterns  
✅ **Performance targets** with caching and optimization strategies  
✅ **Security architecture** including encryption, RBAC, and audit logging  
✅ **Compliance frameworks** for GDPR and data protection  
✅ **Deployment strategies** using Docker, Kubernetes, and CI/CD  
✅ **Code examples** in Node.js/Express with real-world patterns  
✅ **Monitoring and observability** setup with alerts  
✅ **Implementation roadmap** broken into manageable phases  

**Next Steps:**
1. Review and approve technical decisions
2. Prepare development environment (Docker setup)
3. Initialize Prisma migrations
4. Implement Phase 1 endpoints
5. Set up CI/CD pipeline for deployment
6. Begin integration testing

