# Quick Start Guide: Lead Module MVP Development

> **Target Audience:** Backend developers  
> **Duration:** 5-10 minutes to setup  
> **Phase:** MVP Phase 1  

---

## Prerequisites

- **Node.js** 18+ LTS ([download](https://nodejs.org/))
- **MySQL** 8.0+ ([download](https://dev.mysql.com/downloads/mysql/))
- **Git** for version control
- **Docker & Docker Compose** (optional, for containerized setup)
- **Postman** or **Thunder Client** for API testing
- **VS Code** (recommended editor)

---

## Project Setup

### Option 1: Docker Setup (Recommended)

**Fastest way to get started with pre-configured services.**

```bash
# Clone or navigate to project
cd d:/Srithar/Breaking\ Code\ POC/bc-travel-app

# Copy environment file
cp .env.example .env

# Start services (MySQL, Redis, etc.)
docker-compose up -d

# Verify services are running
docker-compose ps

# View logs
docker-compose logs -f api
```

Services will be available at:
- **API:** http://localhost:3000
- **MySQL:** localhost:3306
- **phpMyAdmin:** http://localhost:8080 (admin/admin)

### Option 2: Local Development Setup

**If you prefer local MySQL installation:**

```bash
# 1. Install dependencies
npm install

# 2. Configure environment
cp .env.example .env
# Edit .env:
# DB_HOST=localhost
# DB_USER=root
# DB_PASSWORD=your_password
# DB_NAME=travel_leads_dev

# 3. Run database migrations
npx sequelize-cli db:migrate

# 4. Seed test data (optional)
npx sequelize-cli db:seed:all

# 5. Start development server
npm run dev
```

---

## Project Structure

```
src/
├── config/
│   ├── database.js           # Sequelize configuration
│   └── environment.js        # Environment variables validation
├── middleware/
│   ├── auth.js               # Session authentication
│   ├── authorization.js      # Role-based access control
│   ├── errorHandler.js       # Global error handler
│   └── validators.js         # Input validation schemas
├── routes/
│   ├── leads.routes.js       # Lead endpoints
│   ├── auth.routes.js        # Authentication endpoints
│   └── index.js              # Route aggregator
├── controllers/
│   ├── leadsController.js    # Lead business logic
│   ├── authController.js     # Auth logic
│   └── index.js              # Controller exports
├── models/
│   ├── Organization.js
│   ├── User.js
│   ├── Lead.js
│   ├── Quote.js
│   └── index.js              # Model associations
├── services/
│   ├── leadService.js        # Database operations
│   ├── authService.js        # Auth operations
│   └── auditService.js       # Audit logging
├── utils/
│   ├── logger.js             # Winston logger
│   ├── errorCodes.js         # Error constants
│   ├── validators.js         # Joi validation schemas
│   └── helpers.js            # Utility functions
└── app.js                    # Express app initialization
```

---

## Common Development Tasks

### 1. Create New Migration

```bash
# Generate new migration file
npx sequelize-cli migration:generate --name add_new_field_to_leads

# Run migrations
npm run db:migrate

# Rollback last migration
npm run db:migrate:undo
```

### 2. Add New Endpoint

**Example: Add new endpoint to get lead statistics**

```javascript
// routes/leads.routes.js
router.get('/stats', authMiddleware, leadController.getLeadStats);

// controllers/leadsController.js
async getLeadStats(req, res, next) {
  try {
    const stats = await LeadService.getStats(req.user.org_id);
    res.json({
      success: true,
      data: stats
    });
  } catch (error) {
    next(error);
  }
}
```

### 3. Write Tests

```bash
# Run all tests
npm test

# Run specific test file
npm test -- tests/leads.test.js

# Watch mode (auto-rerun on changes)
npm run test:watch

# Coverage report
npm run test:coverage
```

**Example test:**

```javascript
// tests/leads.test.js
describe('Lead API', () => {
  it('should create a lead', async () => {
    const response = await request(app)
      .post('/api/v1/leads')
      .set('Authorization', `Bearer ${sessionToken}`)
      .send({
        lead_name: 'Test Lead',
        email: 'test@example.com',
        phone: '+919876543210',
        country_code: 'IN',
        travel_date: '2025-03-28',
        destination: 'Andaman',
        travel_type: 'DOMESTIC',
        num_travelers: 5,
        lead_source: 'WEBSITE'
      });

    expect(response.status).toBe(201);
    expect(response.body.success).toBe(true);
    expect(response.body.data.lead_id).toBeDefined();
  });
});
```

### 4. Debug Issues

```bash
# Start with debug logging enabled
DEBUG=* npm run dev

# Or in VS Code, use built-in debugger
# Press F5 to start debugging session
```

### 5. View Database

```bash
# Using phpMyAdmin (Docker)
# Open http://localhost:8080
# Login: admin / admin

# Or using MySQL CLI
mysql -h localhost -u root -p travel_leads_dev
```

---

## API Testing

### Login & Get Session

```bash
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@agency.com",
    "password": "password123"
  }' \
  -c cookies.txt
```

### Create a Lead

```bash
curl -X POST http://localhost:3000/api/v1/leads \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "lead_name": "Mustaq Ahmed",
    "email": "mustaq@example.com",
    "phone": "+919876543210",
    "country_code": "IN",
    "travel_date": "2025-03-28",
    "destination": "Andaman",
    "travel_type": "DOMESTIC",
    "num_travelers": 10,
    "budget": 150000,
    "lead_source": "WEBSITE",
    "notes": "Family trip"
  }'
```

### List Leads

```bash
curl -X GET "http://localhost:3000/api/v1/leads?page=1&limit=10&status=NEW" \
  -b cookies.txt
```

### Search Leads

```bash
curl -X GET "http://localhost:3000/api/v1/leads/search?q=mustaq" \
  -b cookies.txt
```

### Update Lead Status

```bash
curl -X PATCH http://localhost:3000/api/v1/leads/{lead_id}/status \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "new_status": "CONTACTED",
    "reason": "Customer called to confirm itinerary"
  }'
```

### Convert to Quote

```bash
curl -X POST http://localhost:3000/api/v1/leads/{lead_id}/convert-to-quote \
  -H "Content-Type: application/json" \
  -b cookies.txt \
  -d '{
    "quote_note": "Customer approved proposal",
    "estimated_quote_value": 200000
  }'
```

---

## Environment Variables

**Create `.env` file in project root:**

```bash
# Server Configuration
NODE_ENV=development
PORT=3000
LOG_LEVEL=debug

# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=password
DB_NAME=travel_leads_dev

# Redis (optional MVP)
REDIS_HOST=localhost
REDIS_PORT=6379

# Session
SESSION_SECRET=your-secret-key-change-in-production
SESSION_TIMEOUT=1800000  # 30 minutes

# JWT (optional)
JWT_SECRET=your-jwt-secret-key
JWT_EXPIRY=7d

# CORS
CORS_ORIGIN=http://localhost:3000,http://localhost:3001

# Rate Limiting
RATE_LIMIT_WINDOW=60000  # 1 minute
RATE_LIMIT_MAX_REQUESTS=100

# Encryption (Production)
ENCRYPTION_KEY=your-32-char-hex-key-production-only
ENCRYPTION_ALGORITHM=aes-256-gcm
```

---

## Database Seeding

Create test data for development:

```bash
# Run seeders
npx sequelize-cli db:seed:all

# Or specific seeder
npx sequelize-cli db:seed --seed 20260131100000-seed-test-data
```

**Seeder example:**

```javascript
// seeders/20260131100000-seed-test-data.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    const orgId = Sequelize.literal('UUID()');
    
    // Create organization
    await queryInterface.bulkInsert('organizations', [
      {
        org_id: 'f47ac10b-58cc-4372-a567-0e02b2c3d479',
        org_name: 'Test Travel Agency',
        subscription_tier: 'STARTER',
        createdAt: new Date(),
        updatedAt: new Date()
      }
    ]);

    // Create user
    await queryInterface.bulkInsert('users', [
      {
        user_id: '550e8400-e29b-41d4-a716-446655440000',
        org_id: 'f47ac10b-58cc-4372-a567-0e02b2c3d479',
        email: 'admin@agency.com',
        password_hash: '$2b$10$...', // bcrypt hash of 'password123'
        name: 'Admin User',
        role: 'ADMIN',
        is_active: true,
        createdAt: new Date(),
        updatedAt: new Date()
      }
    ]);

    // Create leads
    await queryInterface.bulkInsert('leads', [
      {
        lead_id: '660f9501-f40c-52e5-b668-557765d4e590',
        org_id: 'f47ac10b-58cc-4372-a567-0e02b2c3d479',
        lead_name: 'Mustaq Ahmed',
        email: 'mustaq@example.com',
        phone: '+919876543210',
        country_code: 'IN',
        travel_date: new Date('2025-03-28'),
        destination: 'Andaman',
        travel_type: 'DOMESTIC',
        num_travelers: 10,
        budget: 150000,
        lead_source: 'WEBSITE',
        status: 'NEW',
        created_by: '550e8400-e29b-41d4-a716-446655440000',
        createdAt: new Date(),
        updatedAt: new Date()
      }
    ]);
  },

  down: async (queryInterface) => {
    await queryInterface.bulkDelete('leads', null, {});
    await queryInterface.bulkDelete('users', null, {});
    await queryInterface.bulkDelete('organizations', null, {});
  }
};
```

---

## Useful Commands

```bash
# Development
npm run dev              # Start dev server with hot reload

# Testing
npm test               # Run all tests
npm run test:watch    # Watch mode
npm run test:coverage # Coverage report

# Database
npm run db:migrate     # Run migrations
npm run db:seed        # Seed data
npm run db:reset       # Reset database

# Linting & Formatting
npm run lint           # ESLint check
npm run lint:fix       # Auto-fix lint issues
npm run format         # Prettier formatting

# Production
npm start              # Start production server
npm run build          # Build for production

# Docker
docker-compose up -d         # Start services
docker-compose down          # Stop services
docker-compose logs -f       # View logs
docker exec -it mysql bash   # Access MySQL container
```

---

## Troubleshooting

### MySQL Connection Error

```bash
# Check MySQL is running
docker-compose ps

# View MySQL logs
docker-compose logs mysql

# Restart MySQL
docker-compose restart mysql

# Or locally, restart service
# macOS: brew services restart mysql
# Windows: net stop MySQL80 && net start MySQL80
```

### Migration Failed

```bash
# Rollback migrations
npm run db:migrate:undo:all

# Check migration status
npx sequelize-cli db:migrate:status

# Re-run migrations
npm run db:migrate
```

### Port Already in Use

```bash
# Find process using port 3000
lsof -i :3000  # macOS/Linux
netstat -ano | findstr :3000  # Windows

# Kill process
kill -9 <PID>
```

### Session Not Working

```bash
# Clear cookies
# In browser DevTools → Application → Cookies → Delete all

# Check session in database
mysql> SELECT * FROM sessions;

# Ensure SESSION_SECRET is set in .env
```

---

## Next Steps

1. ✅ Setup local development environment
2. ⏳ Run migrations: `npm run db:migrate`
3. ⏳ Seed test data: `npm run db:seed`
4. ⏳ Start server: `npm run dev`
5. ⏳ Test endpoints with Postman/curl
6. ⏳ Start implementing Phase 1 features
7. ⏳ Run tests: `npm test`
8. ⏳ Deploy to staging

---

## Additional Resources

- **Sequelize Documentation:** https://sequelize.org/docs/v6/
- **Express.js Guide:** https://expressjs.com/
- **MySQL Tutorial:** https://dev.mysql.com/doc/
- **API Documentation:** [API_CONTRACTS.yaml](API_CONTRACTS.yaml)
- **Implementation Plan:** [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md)
- **Data Models:** [DATA_MODELS.md](DATA_MODELS.md)

---

**Questions?** Contact the team or check the [Backend Technical Specification](BACKEND_TECHNICAL_SPECIFICATION.md)

**Version:** 1.0.0  
**Last Updated:** 2026-01-31
