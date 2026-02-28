# Rollback Strategy — Prompt Template for Tech Spec Generation

> **Purpose:** Use this document as a prompt to generate a complete Rollback Strategy Technical Specification for the Transport Management System (TMS). This covers rollback on a **single EC2 instance running Docker Blue-Green** (port-based slot switching via NGINX), database migration rollback, and incident response runbooks.

---

## Prompt Instructions

You are a senior DevOps/SRE engineer. Generate a complete, production-ready Rollback Strategy Technical Specification for the TMS application. Output must be structured Markdown with tables, runbooks, decision trees, and a Gaps section.

---

## Context

- **Application:** Transport Management System (TMS)
- **Stack:** React (Node 24) frontend + Node.js/Express.js (Node 24) backend
- **Deployment Model:** Single EC2 instance, Docker + Docker Compose, Blue-Green via NGINX port switching
- **Blue slot ports:** Backend 3001, Frontend 8081
- **Green slot ports:** Backend 3002, Frontend 8082
- **Active slot tracked in:** `/opt/tms/active-slot` (file contains `blue` or `green`)
- **Pipeline:** GitHub Actions
- **Environments:** DEV | QA | UAT | PROD
- **Registry:** Docker Hub (images tagged `<semver>-<git-sha>`)

---

## Rollback Specification Requirements

### 1. Rollback Trigger Conditions

Generate a complete list of conditions that trigger rollback, categorized by:

**Automatic Triggers (pipeline-driven):**
- Health check failure on Green slot **before** traffic switch → abort deploy, stop Green containers, Blue stays live (this is a pre-switch abort, not a true rollback)
- Health check failure on Green slot **after** traffic switch → rollback: NGINX upstream reverted to Blue ports
- Error rate exceeds threshold post-cutover (e.g. >5% HTTP 5xx within the 10-minute validation window)
- Green containers crash or exit unexpectedly within the validation window

**Manual Triggers (human-initiated):**
- Engineer triggers `workflow_dispatch` rollback workflow in GitHub Actions
- Approval gate rejection during PROD deployment (deploy is cancelled; Blue never left live)
- Post-deployment business validation failure (functional issue not caught by health check)

---

### 2. Rollback Decision Tree

Generate a decision tree (Mermaid flowchart) covering the two distinct scenarios:

**Scenario A — Green fails before traffic switch (pre-switch abort):**
```
Green containers started
  └─ Health checks pass?
       ├─ NO → Stop Green containers (docker compose down)
       │        Blue remains live on NGINX upstream
       │        Write "blue" back to /opt/tms/active-slot (no change)
       │        Notify team → deployment aborted
       └─ YES → Proceed to traffic switch
```

**Scenario B — Green fails after traffic switch (true rollback):**
```
Traffic switched to Green
  └─ Post-switch validation window (10 min)
       ├─ Error spike / health check failure / manual trigger
       │       └─ ROLLBACK:
       │            1. Confirm Blue containers still running
       │            2. Revert NGINX upstream to Blue ports
       │            3. nginx -s reload
       │            4. Verify Blue serving traffic
       │            5. Stop Green containers
       │            6. Write "blue" to /opt/tms/active-slot
       │            7. Notify team
       └─ All clear → Blue teardown → deployment complete
```

---

### 3. Rollback Runbook — Application Layer

**Pre-condition:** Blue containers must still be running on ports 3001 / 8081. They are kept alive throughout the 10-minute validation window for exactly this reason.

| Step | Action | Command / Detail | Automated? | Owner |
|---|---|---|---|---|
| 1 | Confirm Blue containers are running | `docker compose -f /opt/tms/blue/docker-compose.yml ps` | Yes | Pipeline |
| 2 | Health check Blue slot directly | `curl -f http://localhost:3001/api/health` | Yes | Pipeline |
| 3 | Revert NGINX upstream to Blue ports | Update `/opt/tms/nginx/tms.conf` upstream to `127.0.0.1:3001` + `127.0.0.1:8081` | Yes | Pipeline / `switch.sh` |
| 4 | Validate NGINX config | `nginx -t` | Yes | Pipeline |
| 5 | Reload NGINX | `nginx -s reload` | Yes | Pipeline |
| 6 | Verify external traffic via NGINX | `curl -f http://localhost/api/health` (through NGINX) | Yes | Pipeline |
| 7 | Update active slot state file | `echo "blue" > /opt/tms/active-slot` | Yes | Pipeline |
| 8 | Stop and remove Green containers | `docker compose -f /opt/tms/green/docker-compose.yml down` | Yes | Pipeline |
| 9 | Remove failed Green image (tag for audit) | `docker tag <image>:<failed-tag> <image>:failed-<tag>` then optionally remove | Optional | Pipeline |
| 10 | Notify team of rollback | Slack/email alert with version, reason, timestamp, pipeline link | Yes | Pipeline |
| 11 | Tag failed release in Git | `git tag rollback-<version>-<sha>` + push | No | Engineer |
| 12 | Create post-mortem issue | GitHub Issue with deployment context and failure logs | No | Engineer |

---

### 4. What Happens When Blue Is Already Torn Down (Edge Case)

If rollback is triggered **after** the 10-minute validation window has passed and Blue containers have already been torn down:

| Step | Action | Detail |
|---|---|---|
| 1 | Identify previous stable image tag | Read from deployment history or Git tags |
| 2 | Pull previous image from Docker Hub | `docker pull <repo>/tms-backend:<prev-tag>` |
| 3 | Start Blue containers with previous image | `docker compose -f /opt/tms/blue/docker-compose.yml up -d` |
| 4 | Health check Blue slot | Same health check table as above |
| 5 | Switch NGINX back to Blue | Same NGINX switch steps as above |
| 6 | Stop Green containers | `docker compose -f /opt/tms/green/docker-compose.yml down` |

> ⚠️ This scenario takes longer (~3–5 min for image pull + start). To minimise this risk, keep the 10-minute validation window active before tearing down Blue.

---

### 5. Rollback Runbook — Database Layer

> ⚠️ DB rollback must be explicitly reviewed before execution. Never run `Down` commands on PROD without DBA sign-off.

Generate instructions for:
- How to identify which migrations were applied in the failed release
- How to execute `Down` migrations in reverse order
- How to verify DB state post-rollback

**DB Migration Rollback Table Template:**

| Migration ID | Description | Up Command | Down Command | Risk Level | Rollback Safe? | DBA Approval Required? |
|---|---|---|---|---|---|---|
| 001 | Example: Add `tracking_id` column | `ALTER TABLE shipments ADD COLUMN tracking_id VARCHAR(64)` | `ALTER TABLE shipments DROP COLUMN tracking_id` | Low | Yes | No |
| 002 | Example: Drop legacy `notes` column | `ALTER TABLE shipments DROP COLUMN notes` | ⚠️ Data lost — cannot restore | High | No | **YES** |
| 003 | Example: Create new index | `CREATE INDEX idx_shipment_date ON shipments(date)` | `DROP INDEX idx_shipment_date` | Low | Yes | No |

**Risk Classification:**

| Risk | Definition |
|---|---|
| Low | Additive change — fully reversible, no data loss |
| Medium | Schema change that may affect queries but data is preserved |
| High | Destructive change — data loss possible on rollback |
| Critical | Cannot be rolled back — requires point-in-time DB restore |

---

### 6. Environment-Specific Rollback Behaviour

| Environment | Auto-Rollback | Manual Rollback | DB Rollback | Approval Required |
|---|---|---|---|---|
| DEV | Yes | Yes | Best effort | No |
| QA | Yes | Yes | Yes | No |
| UAT | Yes | Yes | Yes | No |
| PROD | Yes (health check + error rate) | Yes (workflow_dispatch) | Yes — DBA review required | **Yes** |

---

### 7. Rollback SLA

| Environment | Detection SLA | Rollback Execution SLA | Communication SLA |
|---|---|---|---|
| DEV | N/A | Best effort | N/A |
| QA | 5 minutes | 10 minutes | Team Slack channel |
| UAT | 5 minutes | 10 minutes | Team + Stakeholder notification |
| PROD | 2 minutes | 5 minutes | Incident channel + On-call page |

---

### 8. GitHub Actions Rollback Workflow

Generate a `rollback.yml` GitHub Actions workflow supporting:
- **Trigger:** `workflow_dispatch` with inputs:
  - `environment` (DEV / QA / UAT / PROD)
  - `rollback_to_version` (semantic version tag to restore)
  - `include_db_rollback` (boolean — default: false)
- **Jobs:**
  1. `validate` — Confirm target image exists in Docker Hub; confirm Blue slot containers are running (or can be restarted from image)
  2. `restore-blue` — If Blue is down: pull previous image + start Blue containers on ports 3001/8081
  3. `health-check-blue` — Verify Blue slot healthy on internal ports before touching NGINX
  4. `switch-traffic` — Revert NGINX upstream to Blue ports + reload
  5. `teardown-green` — Stop and remove Green containers
  6. `db-rollback` (conditional on input) — Run Down migrations in reverse order
  7. `notify` — Alert team with full rollback summary

---

### 9. Blue Slot Retention Policy

| Phase | State | Duration | Action |
|---|---|---|---|
| Post-switch validation | Blue idle, Green live, NGINX on Green | 10 minutes | Keep Blue containers running — available for instant NGINX revert |
| Teardown | No rollback triggered in window | After 10 min | Stop Blue containers: `docker compose down` |
| Emergency restore | Rollback triggered after Blue torn down | N/A | Re-pull previous image from Docker Hub, restart Blue, then switch NGINX |

---

### 10. Rollback Communication Template

```
🔴 ROLLBACK TRIGGERED — [ENVIRONMENT]

Application:    Transport Management System (TMS)
Failed Version: <version>-<sha>  (Green slot — ports 3002/8082)
Rolled Back To: <previous-version>-<sha>  (Blue slot — ports 3001/8081)
Trigger:        <health check failure | error spike | manual>
Time:           <ISO timestamp>
Pipeline Run:   <GitHub Actions URL>

NGINX Status:   Reverted to Blue upstream ✅
Green Status:   Containers stopped ✅
DB Rollback:    <Yes / No / Pending>
Overall Status: <In Progress | Complete | Failed>

Engineer On-Call:  <name>
Post-Mortem Issue: <GitHub Issue URL>
```

---

### 11. Post-Rollback Checklist

- [ ] NGINX upstream confirmed pointing to Blue (ports 3001 / 8081)
- [ ] External health check passing through NGINX
- [ ] `/opt/tms/active-slot` file contains `blue`
- [ ] Green containers stopped and removed
- [ ] Error rate returned to baseline
- [ ] DB state verified (if DB rollback was executed)
- [ ] Team notified via Slack/email
- [ ] Git tag applied: `rollback-<version>-<sha>`
- [ ] Failed image preserved in Docker Hub as `failed-<version>-<sha>` (do not delete — needed for investigation)
- [ ] GitHub Issue created for post-mortem
- [ ] Root cause identified or investigation ticket opened
- [ ] Rollback SLA met — document if breached

---

## Output Requirements

The generated Rollback Strategy Tech Spec must include:
- [ ] Decision tree (Mermaid flowchart) for both pre-switch and post-switch scenarios
- [ ] Application layer rollback runbook (table) with exact commands for port-based NGINX revert
- [ ] Edge case runbook for when Blue containers have already been torn down
- [ ] DB migration rollback table (with risk classification)
- [ ] `rollback.yml` GitHub Actions YAML
- [ ] Blue slot retention policy table
- [ ] Rollback SLA table
- [ ] Post-rollback checklist
- [ ] Notification template
- [ ] Gaps section

---

## Gaps (Pre-filled — Confirm Before Generating)

| Gap | Assumption | Action Required |
|---|---|---|
| Database type | Not confirmed | **Required** — provide DB type to generate migration Down commands |
| Migration tool | Not confirmed | Confirm (Flyway / Liquibase / raw SQL / ORM migrations) |
| DB location | Not confirmed | Confirm — same EC2 or separate RDS/instance |
| Monitoring stack | Not confirmed | Confirm to define error-spike auto-rollback trigger in validation window |
| On-call tooling | Not confirmed | Confirm (PagerDuty / OpsGenie / manual) |
| Post-switch validation window | Assumed 10 minutes | Confirm — this is the window Blue containers are kept warm |
| DB rollback authority | Assumed DBA sign-off required for PROD | Confirm approval chain |
| GitHub Actions runner | GitHub-hosted runner SSH-ing into EC2 | Confirm runner type |