# Implementation Plan: Lead Module MVP - Express.js + MySQL

> **Version:** 1.0.0  
> **Date:** 2026-01-31  
> **Phase:** MVP Phase 1 (1-2 weeks)  
> **Tech Stack:** Node.js + Express.js + MySQL + Redis  
> **Status:** PLANNING

---

## 1. Executive Overview

This document outlines the implementation roadmap for the Lead Module MVP using Express.js and MySQL. The goal is to deliver a production-ready backend API that captures customer travel inquiries, manages lead lifecycle, and enables conversion to quotes.

### Deliverables (MVP Phase 1)

- ✅ Express.js API with complete CRUD operations
- ✅ MySQL database with optimized schema and indexes
- ✅ Authentication & authorization middleware
- ✅ Lead search & filtering with pagination
- ✅ Soft-delete implementation with audit trail
- ✅ Lead-to-quote conversion flow
- ✅ Input validation & error handling
- ✅ Docker setup for local development
- ✅ API documentation (Swagger/OpenAPI)

---

## 2. Technical Context

### Stack Selection Rationale

| Component | Technology | Why Chosen |
|-----------|-----------|-----------|
| **Runtime** | Node.js 18 LTS | Fast, JavaScript ecosystem, easy deployment |
| **Framework** | Express.js 4.x | Lightweight, widely used, minimal overhead |
| **Database** | MySQL 8.0 | Relational data, ACID compliance, proven at scale |
| **ORM/Query** | Sequelize or Raw SQL | Type-safe, migration support |
| **Cache** | Redis (optional MVP) | Session management, rate limiting |
| **Auth** | JWT + HTTP-only cookies | Secure, stateless, standard |
| **Validation** | Joi + express-validator | Comprehensive, readable schemas |
| **Logging** | Winston/Pino | Structured, performance-optimized |
| **Testing** | Jest + Supertest | Comprehensive, industry standard |
| **Container** | Docker | Consistency, easy deployment |

### Architecture Decision Records (ADRs)

**ADR-001: Sequelize ORM vs Raw SQL**
- **Decision:** Use Sequelize with raw SQL fallback for complex queries
- **Rationale:** Better migration management, type safety, community support
- **Trade-off:** Slight performance overhead (mitigated by proper indexing)

**ADR-002: MySQL Single Instance vs Read Replicas**
- **Decision:** Single instance for MVP, read replicas in Phase 2
- **Rationale:** Reduces complexity, sufficient for MVP scale
- **Migration Path:** Easy to add replicas later

**ADR-003: Session Storage Strategy**
- **Decision:** Database-backed sessions (MySQL) for MVP
- **Rationale:** No external dependency, simpler deployment
- **Upgrade Path:** Redis for Phase 2 (performance improvement)

---

## 3. Constitution Check

Based on project constitution principles (from `.specify/memory/constitution.md`):

### Compliance Checklist

- [ ] **Test-First (Non-Negotiable):** TDD approach for all features
- [ ] **Integration Testing:** Focus on contract tests between services
- [ ] **Observability:** Structured logging, request tracing
- [ ] **Versioning:** Semantic versioning (MAJOR.MINOR.PATCH)
- [ ] **Simplicity:** YAGNI principles, no over-engineering

### Implementation Constraints

1. **No external CRM integrations in MVP** (Phase 2+)
2. **Session-based auth only** (no OAuth2 in MVP)
3. **Single-region deployment** (multi-region in Phase 3)
4. **Soft deletes mandatory** (for compliance, audit trail)
5. **All PII must be encrypted** (AES-256 for email, phone in production)

---

## 4. Data Model (MySQL)

### Core Tables (Phase 1)

```
organizations
├─ org_id (UUID, PK)
├─ org_name (VARCHAR 200)
├─ subscription_tier (ENUM: FREE, STARTER, PROFESSIONAL, ENTERPRISE)
└─ timestamps

users
├─ user_id (UUID, PK)
├─ org_id (FK)
├─ email (VARCHAR 254, UNIQUE per org)
├─ password_hash (VARCHAR 255)
├─ role (ENUM: ADMIN, MANAGER, AGENT, VIEWER)
└─ timestamps

leads
├─ lead_id (UUID, PK)
├─ org_id (FK)
├─ lead_name (VARCHAR 100)
├─ email (VARCHAR 254) [ENCRYPTED in production]
├─ phone (VARCHAR 20) [ENCRYPTED in production]
├─ country_code (CHAR 2)
├─ travel_date (DATE)
├─ destination (VARCHAR 100)
├─ travel_type (ENUM: DOMESTIC, INTERNATIONAL)
├─ num_travelers (INT)
├─ budget (DECIMAL 12,2, nullable)
├─ lead_source (ENUM: WEBSITE, PHONE, EMAIL, REFERRAL, SOCIAL, OTHER)
├─ status (ENUM: NEW, CONTACTED, QUOTED, NEGOTIATING, BOOKED, REJECTED, ARCHIVED, CLOSED)
├─ assigned_to (FK users.user_id, nullable)
├─ created_by (FK users.user_id)
├─ notes (TEXT, nullable)
├─ last_contacted_at (TIMESTAMP, nullable)
├─ created_at (TIMESTAMP)
├─ updated_at (TIMESTAMP)
└─ deleted_at (TIMESTAMP, nullable) [For soft deletes]

quotes
├─ quote_id (UUID, PK)
├─ lead_id (FK leads.lead_id, UNIQUE)
├─ org_id (FK organizations.org_id)
├─ status (ENUM: PENDING_APPROVAL, SENT, ACCEPTED, REJECTED, EXPIRED)
├─ total_amount (DECIMAL 12,2, nullable)
├─ created_at (TIMESTAMP)
├─ updated_at (TIMESTAMP)
└─ expires_at (TIMESTAMP)

audit_logs
├─ audit_id (BIGINT, PK, auto_increment)
├─ org_id (FK)
├─ entity_type (VARCHAR 50: LEAD, QUOTE, USER)
├─ entity_id (VARCHAR 36)
├─ action (VARCHAR 50: CREATE, UPDATE, DELETE, STATUS_CHANGE, CONVERT)
├─ old_value (JSON, nullable)
├─ new_value (JSON, nullable)
├─ changed_by (FK users.user_id)
├─ ip_address (VARCHAR 45, nullable)
├─ user_agent (VARCHAR 500, nullable)
├─ request_id (UUID, nullable)
├─ changed_at (TIMESTAMP)
└─ retention_until (TIMESTAMP + 5 years)

sessions (for session management)
├─ sid (VARCHAR 255, PK)
├─ expires (DATETIME)
├─ data (JSON)
└─ created_at (TIMESTAMP)
```

### Indexes (Performance Optimization)

```sql
-- Leads table indexes
CREATE INDEX idx_leads_org_status ON leads(org_id, status) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_org_created ON leads(org_id, created_at DESC) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_assigned_user ON leads(assigned_to, status) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_travel_date ON leads(travel_date) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_destination ON leads(destination) WHERE deleted_at IS NULL;
CREATE INDEX idx_leads_email_phone ON leads(email, phone) WHERE deleted_at IS NULL;

-- Audit logs indexes
CREATE INDEX idx_audit_org_entity ON audit_logs(org_id, entity_type, entity_id);
CREATE INDEX idx_audit_changed_at ON audit_logs(changed_at DESC);
CREATE INDEX idx_audit_changed_by ON audit_logs(changed_by);

-- Users indexes
CREATE INDEX idx_users_org_role ON users(org_id, role) WHERE is_active = 1;

-- Quotes indexes
CREATE INDEX idx_quotes_lead_id ON quotes(lead_id);
CREATE INDEX idx_quotes_org_status ON quotes(org_id, status);
```

---

## 5. API Specification (OpenAPI 3.0)

All endpoints follow RESTful conventions with standard JSON responses.

### Base URL
```
http://localhost:3000/api/v1
```

### Response Format

**Success Response:**
```json
{
  "success": true,
  "data": { /* response data */ },
  "meta": {
    "request_id": "uuid",
    "timestamp": "ISO-8601",
    "response_time_ms": 145
  }
}
```

**Error Response:**
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": []
  },
  "meta": {
    "request_id": "uuid",
    "timestamp": "ISO-8601"
  }
}
```

### Leads Endpoints

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| POST | `/leads` | Create new lead | Required |
| GET | `/leads` | List leads (paginated) | Required |
| GET | `/leads/{lead_id}` | Get lead details | Required |
| PUT | `/leads/{lead_id}` | Update lead | Required |
| PATCH | `/leads/{lead_id}/status` | Update lead status | Required |
| DELETE | `/leads/{lead_id}` | Soft delete lead | Admin only |
| POST | `/leads/{lead_id}/convert-to-quote` | Convert lead to quote | Required |
| GET | `/leads/search` | Full-text search | Required |

### Authentication Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/login` | Login with email/password |
| POST | `/auth/logout` | Logout (clear session) |
| GET | `/auth/me` | Get current user profile |
| POST | `/auth/refresh` | Refresh session |

### Admin Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/admin/organizations` | List organizations |
| POST | `/admin/organizations` | Create organization |
| GET | `/admin/users` | List users |
| POST | `/admin/users` | Create user |

---

## 6. Database Migrations

### Using Sequelize Migrations

```bash
# Generate migration
npx sequelize migration:generate --name create-organizations-table

# Run migrations
npm run db:migrate

# Rollback
npm run db:migrate:undo
```

### Migration Files Structure
```
migrations/
├── 20260131100000-create-organizations.js
├── 20260131100001-create-users.js
├── 20260131100002-create-leads.js
├── 20260131100003-create-quotes.js
├── 20260131100004-create-audit-logs.js
└── 20260131100005-create-sessions.js
```

---

## 7. Project Structure

```
bc-travel-app/
├── src/
│   ├── config/
│   │   ├── database.js          (Sequelize config)
│   │   ├── redis.js             (Redis client - optional MVP)
│   │   └── environment.js       (ENV validation)
│   │
│   ├── middleware/
│   │   ├── auth.js              (Session/JWT validation)
│   │   ├── authorization.js     (Role-based access)
│   │   ├── errorHandler.js      (Global error handler)
│   │   ├── requestLogger.js     (Request logging)
│   │   ├── rateLimit.js         (Rate limiting)
│   │   ├── cors.js              (CORS configuration)
│   │   ├── csrf.js              (CSRF protection)
│   │   └── validators.js        (Input validation)
│   │
│   ├── routes/
│   │   ├── index.js             (Route aggregation)
│   │   ├── leads.routes.js      (Lead endpoints)
│   │   ├── quotes.routes.js     (Quote endpoints)
│   │   ├── auth.routes.js       (Auth endpoints)
│   │   └── admin.routes.js      (Admin endpoints)
│   │
│   ├── controllers/
│   │   ├── leadsController.js   (Lead logic)
│   │   ├── quotesController.js  (Quote logic)
│   │   ├── authController.js    (Auth logic)
│   │   └── adminController.js   (Admin logic)
│   │
│   ├── models/
│   │   ├── Organization.js
│   │   ├── User.js
│   │   ├── Lead.js
│   │   ├── Quote.js
│   │   ├── AuditLog.js
│   │   └── index.js             (Model associations)
│   │
│   ├── services/
│   │   ├── leadService.js       (Business logic)
│   │   ├── quoteService.js
│   │   ├── authService.js
│   │   ├── auditService.js
│   │   └── validationService.js
│   │
│   ├── utils/
│   │   ├── logger.js            (Winston logger)
│   │   ├── errorCodes.js        (Error constants)
│   │   ├── encryption.js        (PII encryption)
│   │   ├── validators.js        (Joi schemas)
│   │   └── helpers.js           (Utility functions)
│   │
│   └── app.js                   (Express app initialization)
│
├── migrations/
│   ├── 20260131100000-create-organizations.js
│   ├── 20260131100001-create-users.js
│   ├── 20260131100002-create-leads.js
│   ├── 20260131100003-create-quotes.js
│   ├── 20260131100004-create-audit-logs.js
│   └── 20260131100005-create-sessions.js
│
├── seeders/
│   └── 20260131100000-seed-test-data.js
│
├── tests/
│   ├── unit/
│   │   ├── services/
│   │   └── utils/
│   ├── integration/
│   │   ├── leads.test.js
│   │   ├── quotes.test.js
│   │   └── auth.test.js
│   └── setup.js                 (Test configuration)
│
├── docs/
│   ├── API.md                   (API documentation)
│   ├── ARCHITECTURE.md          (Architecture notes)
│   └── DEVELOPMENT.md           (Dev setup guide)
│
├── .env.example
├── .env.production
├── .dockerignore
├── docker-compose.yml
├── Dockerfile
├── package.json
├── .sequelizerc                 (Sequelize config file)
└── server.js                    (Entry point)
```

---

## 8. Implementation Phases

### Phase 1: Core MVP (Days 1-5)

**Day 1-2: Setup & Database**
- Initialize Node.js + Express project
- Setup Sequelize ORM
- Create database migrations
- Setup environment configuration
- Docker setup for local dev

**Day 2-3: Authentication & Middleware**
- Implement session-based auth
- Create RBAC middleware
- Error handling middleware
- Request logging setup
- CSRF protection

**Day 3-4: Lead CRUD Endpoints**
- Create lead endpoint with validation
- List leads with pagination
- Get lead details
- Update lead endpoint
- Soft delete endpoint
- Full-text search

**Day 4-5: Advanced Features & Testing**
- Lead status update with transitions
- Convert lead to quote flow
- Audit logging for all operations
- Unit tests (>70% coverage)
- Integration tests
- API documentation (Swagger)

### Phase 1 Deliverables

- ✅ All API endpoints functional
- ✅ Database schema with migrations
- ✅ Authentication & authorization working
- ✅ Audit trail logging
- ✅ Error handling & validation
- ✅ Test coverage >70%
- ✅ Docker setup
- ✅ Swagger/OpenAPI docs
- ✅ README with setup instructions

---

## 9. Key Features Implementation Details

### 9.1 Lead Creation Flow

**Validation:**
1. Email format validation (RFC 5322)
2. Phone E.164 format validation
3. Travel date >= today check
4. Budget positive if provided
5. Num travelers 1-500 range
6. Check duplicate (email + phone combo per org)

**Flow:**
1. Validate inputs
2. Check duplicates
3. Insert into leads table
4. Log audit entry (CREATE)
5. Return 201 with lead object

### 9.2 Lead Listing with Filters

**Query Parameters:**
- `page` (default: 1)
- `limit` (default: 25, max: 100)
- `status` (NEW, CONTACTED, QUOTED, etc.)
- `search` (name, email, phone)
- `travel_type` (DOMESTIC, INTERNATIONAL)
- `date_from` and `date_to` (travel date range)
- `destination` (exact match or fuzzy)
- `assigned_to` (user ID)
- `sort_by` (created_at, travel_date, name)
- `sort_order` (ASC, DESC)

**Access Rules:**
- ADMIN: See all leads for org
- MANAGER: See leads assigned to team
- AGENT: See only assigned leads
- VIEWER: See all leads (read-only)

### 9.3 Lead Status Transitions

```
NEW → CONTACTED (by agent/manager/admin)
    ↓
CONTACTED → QUOTED (by agent/manager/admin)
    ↓
QUOTED → NEGOTIATING (by agent/manager/admin)
    ↓
NEGOTIATING → BOOKED (by agent/manager/admin)
    ↓
BOOKED → CLOSED (by admin)

Any status → REJECTED (by agent/manager/admin)
Any status → ARCHIVED (automatic after 90 days no contact, or manual)

Restrictions:
- Cannot convert already CONVERTED leads
- Cannot delete BOOKED leads (soft delete only)
- Cannot change status of CLOSED/ARCHIVED leads
```

### 9.4 Convert Lead to Quote

**Atomic Transaction:**
1. BEGIN TRANSACTION
2. Verify lead exists & is in QUOTED/NEGOTIATING status
3. Create quote record linked to lead
4. Update lead status to CONVERTED
5. Create audit log entry
6. COMMIT TRANSACTION
7. Emit event for notifications (Phase 2)

**Error Handling:**
- Lead not found → 404
- Already converted → 409 Conflict
- Invalid status for conversion → 400
- Permission denied → 403

---

## 10. Error Handling Strategy

### Standard Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| `VALIDATION_ERROR` | 400 | Input validation failed |
| `UNAUTHORIZED` | 401 | Session expired/invalid |
| `PERMISSION_DENIED` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `CONFLICT` | 409 | Business logic conflict |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Input validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  }
}
```

---

## 11. Security Considerations

### Authentication
- Session-based using HTTP-only, Secure, SameSite=Strict cookies
- Session timeout: 30 minutes
- Password hashing: bcrypt with salt rounds 10+

### Authorization
- Role-based access control (RBAC)
- Ownership checks (agents only access own leads)
- Organizational isolation (no cross-org data access)

### Data Protection
- PII encryption in production (AES-256)
- TLS 1.3+ for all transit
- Input sanitization to prevent SQL injection
- XSS prevention through output encoding

### Audit & Compliance
- Complete audit trail for all changes
- User IP & user agent tracking
- GDPR compliance (right to erasure)
- Data retention policies enforced

---

## 12. Performance Targets

### Latency SLAs (MVP)

| Operation | Target | Max |
|-----------|--------|-----|
| Create Lead | <200ms | <500ms |
| List Leads (p1) | <500ms | <1000ms |
| Search Leads | <500ms | <1000ms |
| Get Lead Details | <300ms | <700ms |
| Update Lead | <200ms | <500ms |
| Update Status | <150ms | <400ms |
| Convert to Quote | <300ms | <800ms |

### Scalability Targets

- 100+ concurrent users
- 5,000 leads/day creation
- 500 leads/hour average
- Support for 1M+ leads (with archiving)

### Database Optimization

- Connection pooling: 10-20 connections
- Proper indexes on frequently queried columns
- Query optimization (no N+1 queries)
- Pagination enforced (no unlimited queries)
- Archive old leads to separate table (Phase 2)

---

## 13. Testing Strategy

### Unit Tests (>70% coverage)
- Service layer business logic
- Utility functions (validators, encryption, etc.)
- Error handling paths
- Framework: Jest

### Integration Tests
- API endpoint tests (request/response)
- Database operations
- Transaction rollback scenarios
- Authorization checks

### End-to-End Tests (Phase 2)
- Complete lead workflows
- Multi-step operations
- Error recovery

### Test Execution

```bash
npm run test              # Run all tests
npm run test:watch       # Watch mode
npm run test:coverage    # Coverage report
npm run test:integration # Integration tests only
```

---

## 14. Deployment & DevOps

### Local Development Setup

```bash
# Copy environment
cp .env.example .env

# Install dependencies
npm install

# Start services (Docker Compose)
docker-compose up -d

# Run migrations
npm run db:migrate

# Seed test data
npm run db:seed

# Start server
npm run dev
```

### Docker Configuration

**Dockerfile:**
- Multi-stage build for optimized size
- Node 18-alpine base image
- Non-root user execution
- Health checks configured

**docker-compose.yml:**
- Express API service
- MySQL database
- Redis (optional MVP)
- pgAdmin for database inspection

### CI/CD Pipeline (Phase 2)

- GitHub Actions for testing
- Automated deployment to staging
- Manual approval for production
- Rollback capability

---

## 15. Documentation Requirements

### API Documentation (Swagger/OpenAPI)
- Complete endpoint definitions
- Request/response schemas
- Error codes and examples
- Authentication requirements

### Developer Documentation
- Architecture overview
- Database schema explanation
- Setup and development guide
- Common tasks and troubleshooting

### Operations Documentation
- Deployment procedures
- Backup & recovery plans
- Monitoring & alerting setup
- Troubleshooting guide

---

## 16. Known Risks & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| Database schema changes | Medium | High | Use migrations, backup before changes |
| Concurrent data edits | Low | Medium | Implement optimistic locking (Phase 2) |
| PII data exposure | Low | Critical | Encryption, audit logs, access control |
| Performance degradation | Medium | High | Indexing, caching strategy, monitoring |
| Cross-org data leak | Low | Critical | Org isolation checks, tests verify |

---

## 17. Success Criteria

- ✅ All MVP features implemented and working
- ✅ Test coverage >70% (unit + integration)
- ✅ Performance benchmarks met (p95 latency <1s)
- ✅ No data loss during system failures
- ✅ Zero security vulnerabilities (OWASP Top 10)
- ✅ 99% uptime in staging
- ✅ API documented (Swagger)
- ✅ Team can deploy independently

---

## 18. Timeline & Milestones

```
Week 1:
├─ Day 1 (Jan 31): Setup & Database Design
├─ Day 2 (Feb 1):  Sequelize Migrations & Auth
├─ Day 3 (Feb 2):  Lead CRUD Endpoints
├─ Day 4 (Feb 3):  Search & Filtering
├─ Day 5 (Feb 4):  Testing & Documentation

Week 2 (Buffer):
├─ Day 1 (Feb 7):  Code Review & Refinement
├─ Day 2 (Feb 8):  Performance Optimization
├─ Day 3 (Feb 9):  Security Audit
├─ Day 4 (Feb 10): Final Testing
└─ Day 5 (Feb 11): Deployment & Handoff
```

---

## 19. Next Steps

1. ✅ Review and approve this implementation plan
2. ⏳ Create detailed API contracts (OpenAPI/Swagger)
3. ⏳ Generate data models and migrations
4. ⏳ Initialize Express.js project structure
5. ⏳ Implement Phase 1 endpoints (Week 1)
6. ⏳ Execute testing & documentation
7. ⏳ Deploy to staging environment

---

**Document Version:** 1.0.0  
**Last Updated:** 2026-01-31  
**Status:** READY FOR IMPLEMENTATION
