# Monitoring & Logging Specification — Transport Management System (TMS)

> **Version:** 1.0.0  
> **Date:** 2026-02-28  
> **Stack:** Backend — Node.js + Express.js | Frontend — React  
> **Platform:** AWS (Backend — Direct server / PM2, Frontend — Amplify)

---

## 1. Monitoring Tools

| Layer | Tool | Purpose | Environment |
|---|---|---|---|
| **Backend APM** | Elastic APM (Node.js Agent) | Request tracing, error tracking, transaction metrics, dependency mapping | DEV, QA, UAT, PROD |
| **Frontend Analytics** | Google Analytics (GA4) | User behavior, page views, session metrics, conversion funnels | QA, UAT, PROD |
| **Infrastructure** | AWS CloudWatch | EC2 instance metrics (CPU, Memory, Disk, Network), alarms | All |
| **Uptime** | AWS CloudWatch Synthetics / UptimeRobot | Endpoint health checks, SSL certificate monitoring | UAT, PROD |
| **Log Aggregation** | Elastic Stack (ELK) — Elasticsearch + Logstash + Kibana | Centralized log ingestion, search, and visualization | All |
| **Alerting** | Elastic Alerting + AWS CloudWatch Alarms | Threshold-based and anomaly-based alerts | UAT, PROD |

---

## 2. Application Metrics

### 2.1 Backend Metrics (Elastic APM)

| Metric | Description | Unit |
|---|---|---|
| `transaction.duration` | End-to-end response time per API endpoint | ms |
| `transaction.throughput` | Requests per second (RPS) per endpoint | req/s |
| `error.rate` | Percentage of failed transactions (4xx/5xx) | % |
| `span.db.duration` | Database query execution time | ms |
| `span.external.duration` | External HTTP call latency (e.g., payment gateways, APIs) | ms |
| `memory.heap.used` | Node.js heap memory usage | MB |
| `cpu.process.percent` | Node.js process CPU utilization | % |
| `event_loop.delay` | Node.js event loop latency | ms |
| `gc.time` | Garbage collection time | ms |
| `active_handles` | Active handles/connections in the process | count |

### 2.2 Frontend Metrics (Google Analytics GA4)

| Metric | Description | Unit |
|---|---|---|
| `page_view` | Page views per route/page | count |
| `session_duration` | Average session duration | seconds |
| `first_contentful_paint` | Time to first meaningful render | ms |
| `largest_contentful_paint` | Time to largest visible content render | ms |
| `cumulative_layout_shift` | Visual stability score | score |
| `first_input_delay` | Time to first interactivity | ms |
| `bounce_rate` | Percentage of single-page sessions | % |
| `custom_event.*` | Business events (bookings, searches, form submissions) | count |

### 2.3 Infrastructure Metrics (CloudWatch)

| Metric | Description | Unit |
|---|---|---|
| `CPUUtilization` | EC2 instance CPU usage | % |
| `MemoryUtilization` | EC2 instance memory usage (via CloudWatch Agent) | % |
| `DiskSpaceUtilization` | Disk usage on server volumes | % |
| `NetworkIn / NetworkOut` | Network I/O bandwidth | bytes |
| `StatusCheckFailed` | Instance health status checks | count |

---

## 3. Alert Thresholds

### 3.1 Critical Alerts (P1 — Immediate Action Required)

| Alert Name | Condition | Threshold | Channel | Escalation |
|---|---|---|---|---|
| **API Down** | Health check endpoint returns non-200 for ≥ 2 consecutive checks | 2 failures in 2 min | Slack `#tms-alerts` + PagerDuty | On-call engineer → Team lead (5 min) |
| **Error Rate Spike** | 5xx error rate exceeds threshold | > 5% over 5 min | Slack `#tms-alerts` + PagerDuty | On-call engineer → Team lead (5 min) |
| **CPU Critical** | EC2 CPU utilization sustained high | > 90% for 5 min | Slack `#tms-infra` + CloudWatch Alarm | Infra team → DevOps lead (10 min) |
| **Memory Critical** | EC2 or Node.js memory usage critical | > 90% for 5 min | Slack `#tms-infra` + CloudWatch Alarm | Infra team → DevOps lead (10 min) |
| **Disk Full** | Disk utilization critical | > 90% | Slack `#tms-infra` + CloudWatch Alarm | Infra team |

### 3.2 Warning Alerts (P2 — Investigate Within 30 Min)

| Alert Name | Condition | Threshold | Channel |
|---|---|---|---|
| **High Latency** | Average API response time elevated | > 2000 ms over 5 min | Slack `#tms-alerts` |
| **Elevated Error Rate** | 4xx/5xx error rate above normal | > 2% over 10 min | Slack `#tms-alerts` |
| **CPU Warning** | CPU utilization elevated | > 70% for 10 min | Slack `#tms-infra` |
| **Memory Warning** | Memory usage elevated | > 75% for 10 min | Slack `#tms-infra` |
| **Event Loop Lag** | Node.js event loop delay high | > 100 ms | Slack `#tms-alerts` |
| **Disk Warning** | Disk utilization elevated | > 75% | Slack `#tms-infra` |

### 3.3 Informational Alerts (P3 — Review During Business Hours)

| Alert Name | Condition | Threshold | Channel |
|---|---|---|---|
| **Deployment Completed** | CI/CD pipeline deploys to any environment | On every deploy | Slack `#tms-deployments` |
| **Daily Error Summary** | Aggregated daily error report | Scheduled at 09:00 UTC | Slack `#tms-alerts` |
| **Traffic Anomaly** | Traffic deviates from baseline | ±50% from 7-day avg | Slack `#tms-alerts` |

---

## 4. Centralized Logging Strategy

### 4.1 Architecture

```
┌─────────────────┐     ┌─────────────┐     ┌───────────────┐     ┌─────────┐
│  Node.js App    │────▶│  Filebeat   │────▶│  Logstash     │────▶│ Elastic │
│  (Winston/Pino) │     │  (shipper)  │     │  (transform)  │     │ Search  │
└─────────────────┘     └─────────────┘     └───────────────┘     └────┬────┘
                                                                       │
┌─────────────────┐                                                    ▼
│  React App      │──── GA4 + Browser Console ───▶              ┌─────────┐
│  (Frontend)     │                                             │ Kibana  │
└─────────────────┘                                             └─────────┘
```

### 4.2 Log Levels

| Level | Usage | Example |
|---|---|---|
| `error` | Application errors, uncaught exceptions, failed operations | DB connection failure, payment API timeout |
| `warn` | Degraded behavior, deprecated usage, recoverable issues | Retry attempt, rate limit approaching |
| `info` | Business events, request lifecycle, state transitions | User login, booking created, deployment started |
| `debug` | Detailed diagnostic info (DEV/QA only) | Request payload, query parameters, cache hits |
| `trace` | Granular tracing (DEV only) | Function entry/exit, variable state |

### 4.3 Log Format (Structured JSON)

All backend logs MUST use structured JSON format for machine-parseable ingestion:

```json
{
  "timestamp": "2026-02-28T12:00:00.000Z",
  "level": "info",
  "service": "tms-backend",
  "environment": "PROD",
  "traceId": "abc123def456",
  "spanId": "span789",
  "message": "Booking created successfully",
  "userId": "user_42",
  "bookingId": "BK-20260228-001",
  "duration": 234,
  "metadata": {
    "route": "POST /api/bookings",
    "statusCode": 201,
    "ip": "10.0.1.55"
  }
}
```

### 4.4 Log Categories

| Category | Description | Index Pattern |
|---|---|---|
| **Application Logs** | API requests, business logic, errors | `tms-app-{env}-*` |
| **Access Logs** | HTTP request/response metadata | `tms-access-{env}-*` |
| **Audit Logs** | Security-sensitive actions (see §5) | `tms-audit-{env}-*` |
| **System Logs** | PM2 process logs, OS-level events | `tms-system-{env}-*` |
| **Performance Logs** | Slow queries, high-latency requests | `tms-perf-{env}-*` |

### 4.5 Environment-Specific Log Configuration

| Setting | DEV | QA | UAT | PROD |
|---|---|---|---|---|
| **Log Level** | `debug` | `debug` | `info` | `info` |
| **Console Output** | ✅ Yes | ✅ Yes | ❌ No | ❌ No |
| **File Output** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **ELK Shipping** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **APM Sampling** | 100% | 100% | 50% | 10% |
| **Log Rotation** | 50 MB / 3 files | 100 MB / 5 files | 200 MB / 7 files | 500 MB / 14 files |

---

## 5. Audit Log Retention Policy

### 5.1 Auditable Events

| Event Category | Actions Logged | Data Captured |
|---|---|---|
| **Authentication** | Login, Logout, Failed login, Password reset, Token refresh | userId, IP, userAgent, timestamp, result |
| **Authorization** | Role change, Permission grant/revoke | actorId, targetUserId, oldRole, newRole |
| **Data Mutation** | Create, Update, Delete on business entities | actorId, entity, entityId, changedFields (before/after) |
| **Configuration** | Environment variable change, Feature flag toggle | actorId, setting, oldValue, newValue |
| **Admin Actions** | User suspension, Data export, Bulk operations | actorId, action, scope, affectedCount |
| **API Access** | External API key usage, Rate limit hits | apiKeyId, endpoint, responseCode |

### 5.2 Audit Log Format

```json
{
  "timestamp": "2026-02-28T12:00:00.000Z",
  "eventType": "DATA_MUTATION",
  "action": "UPDATE",
  "actor": {
    "userId": "user_42",
    "role": "admin",
    "ip": "10.0.1.55",
    "userAgent": "Mozilla/5.0..."
  },
  "resource": {
    "entity": "Booking",
    "entityId": "BK-20260228-001"
  },
  "changes": {
    "before": { "status": "pending" },
    "after": { "status": "confirmed" }
  },
  "result": "success",
  "correlationId": "req-abc-123"
}
```

### 5.3 Retention Schedule

| Log Type | DEV | QA | UAT | PROD | Compliance Note |
|---|---|---|---|---|---|
| **Application Logs** | 7 days | 14 days | 30 days | 90 days | — |
| **Access Logs** | 7 days | 14 days | 30 days | 90 days | — |
| **Audit Logs** | 30 days | 30 days | 90 days | **1 year** | Regulatory / compliance requirement |
| **Performance Logs** | 7 days | 14 days | 30 days | 60 days | — |
| **System Logs** | 7 days | 14 days | 30 days | 60 days | — |
| **Error/Crash Dumps** | 14 days | 30 days | 60 days | 180 days | For post-incident analysis |

### 5.4 Retention Implementation

| Action | Tool | Trigger |
|---|---|---|
| **Index Lifecycle Management (ILM)** | Elasticsearch ILM Policy | Automated — time-based rollover |
| **Hot → Warm → Cold → Delete** | Elasticsearch ILM Phases | Hot: 7 days → Warm: 30 days → Cold: remaining → Delete at retention limit |
| **Archival (PROD Audit Logs)** | S3 Glacier Deep Archive | After 90 days active, archive to S3 for remaining retention |
| **Local Log Rotation** | PM2 `pm2-logrotate` module | Max file size + max retained files per environment |

---

## 6. Implementation Checklist

### 6.1 Backend Setup

- [ ] Install Elastic APM Node.js agent (`elastic-apm-node`)
- [ ] Configure structured logging library (Winston or Pino) with JSON format
- [ ] Create audit log middleware for Express.js routes
- [ ] Add correlation ID middleware (trace propagation)
- [ ] Configure Filebeat on server to ship logs to Logstash/Elasticsearch
- [ ] Set up PM2 log rotation (`pm2-logrotate`)
- [ ] Add health check endpoint (`GET /healthz`) for uptime monitoring
- [ ] Configure `.env` variables per environment for log levels and APM settings

### 6.2 Frontend Setup

- [ ] Integrate Google Analytics GA4 tracking script
- [ ] Configure custom events for business actions (bookings, searches)
- [ ] Set up Core Web Vitals reporting (LCP, FID, CLS)
- [ ] Add error boundary logging for React component crashes

### 6.3 Infrastructure Setup

- [ ] Deploy Elastic Stack (Elasticsearch + Logstash + Kibana) or use Elastic Cloud
- [ ] Install CloudWatch Agent on EC2 instances for memory/disk metrics
- [ ] Create CloudWatch Alarms for infrastructure thresholds
- [ ] Configure Elasticsearch ILM policies per log category
- [ ] Set up S3 bucket + lifecycle rules for audit log archival
- [ ] Create Kibana dashboards (Application Overview, Error Analysis, Audit Trail)

### 6.4 Alerting Setup

- [ ] Configure Elastic Alerting rules for application-level alerts
- [ ] Configure CloudWatch Alarms for infrastructure-level alerts
- [ ] Set up Slack webhook integration for alert channels
- [ ] Define escalation policies and on-call rotation
- [ ] Test alert pipeline end-to-end in QA before PROD rollout

---

## 7. Kibana Dashboard Specifications

| Dashboard | Panels | Refresh Rate |
|---|---|---|
| **Application Overview** | RPS, Avg Response Time, Error Rate, Top Endpoints, Active Users | 30s |
| **Error Analysis** | Error count by type, Error trend (24h), Stack trace viewer, Affected users | 1 min |
| **Infrastructure Health** | CPU, Memory, Disk, Network I/O, Process Count | 30s |
| **Audit Trail** | Recent audit events, Filter by user/action/entity, Timeline view | 5 min |
| **Deployment Tracker** | Recent deployments, Post-deploy error delta, Rollback history | 5 min |

---

## 8. Environment Variables

The following `.env` variables control monitoring and logging behavior per environment:

```env
# Elastic APM
ELASTIC_APM_SERVICE_NAME=tms-backend
ELASTIC_APM_SERVER_URL=https://apm.example.com:8200
ELASTIC_APM_SECRET_TOKEN=<from AWS Secrets Manager>
ELASTIC_APM_ENVIRONMENT=PROD
ELASTIC_APM_TRANSACTION_SAMPLE_RATE=0.1

# Logging
LOG_LEVEL=info
LOG_FORMAT=json
LOG_FILE_PATH=/var/log/tms/app.log
LOG_MAX_SIZE=500m
LOG_MAX_FILES=14

# Google Analytics
REACT_APP_GA_MEASUREMENT_ID=G-XXXXXXXXXX

# Alerting
ALERT_SLACK_WEBHOOK_URL=<from AWS Secrets Manager>
ALERT_PAGERDUTY_ROUTING_KEY=<from AWS Secrets Manager>
```

---

## Gaps

> [!IMPORTANT]
> The following items need confirmation before finalizing the monitoring setup:

- **Database type not confirmed** — Specific DB metrics (e.g., PostgreSQL `pg_stat`, MongoDB profiler) cannot be defined until the database engine is confirmed.
- **Elastic Stack hosting** — Confirm if Elastic Cloud (managed) or self-hosted ELK will be used. This impacts infrastructure sizing and maintenance.
- **PagerDuty / On-call tool** — Confirm if PagerDuty or another incident management tool is in use for escalation routing.
- **Compliance requirements** — Confirm if there are specific regulatory requirements (GDPR, SOC2, PCI-DSS) that mandate longer audit log retention or specific log content.
- **Budget for Elastic Cloud** — If using managed Elastic Cloud, confirm the tier/plan to determine data retention limits and ingestion quotas.
- **Slack workspace and channel names** — Confirm actual Slack channel names for alert routing.
