## Deployment Strategy

| Service | Platform | Strategy |
|---|---|---|
| **Frontend (React)** | **AWS Amplify** | Amplify managed CI/CD — branch-based auto-deploy |
| **Backend (Node.js/Express)** | **EC2 — Ubuntu 22.04** | Single-machine Docker Blue-Green |

---

## Frontend — AWS Amplify Deployment

### How it works

The React frontend is deployed via **AWS Amplify**. Amplify watches the configured branches and automatically builds + deploys on every push or merge. No Docker, no NGINX, no PM2 required for the frontend.

### Amplify Branch → Environment Mapping

| Branch | Amplify Environment | URL |
|---|---|---|
| `dev/*` | DEV | `dev.tms.example.com` |
| `test` | QA | `qa.tms.example.com` |
| `uat/*` | UAT | `uat.tms.example.com` |
| `prod` | PROD | `tms.example.com` |

### Amplify Build Spec (`amplify.yml`)

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - cd ui
        - npm ci --prefer-offline
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: ui/build
    files:
      - '**/*'
  cache:
    paths:
      - ui/node_modules/**/*
```

### Amplify Environment Variables
- Sensitive values pulled from **AWS Secrets Manager** via Amplify's secret injection
- `REACT_APP_API_URL` points to the backend EC2 endpoint per environment

### Amplify Rollback
- Navigate to **Amplify Console → App → Branch → Deployments**
- Select the previous successful deployment
- Click **Redeploy this version** — zero-downtime instant rollback

---

## Backend — Single-Machine Docker Blue-Green

### Core Concept

On a single EC2 machine, Blue and Green are **two Docker containers running on different internal ports simultaneously**. NGINX acts as the router, pointing all external traffic to whichever slot is currently active. The inactive slot runs quietly in the background until it is either promoted or torn down.

```
Internet
   │
   ▼
NGINX (port 80 / 443)
   │
   ├── Active slot → Blue containers  (e.g. backend :3001, frontend :8081)
   │                      OR
   └── Active slot → Green containers (e.g. backend :3002, frontend :8082)

Both sets of containers run on the same EC2 host.
Only one slot receives external traffic at a time.
```

### Slot Port Assignment (Backend only)

> **Frontend is on AWS Amplify** — no port assignment needed for UI.

| Slot | Backend Internal Port |
|---|---|
| Blue | 3001 |
| Green | 3002 |

NGINX upstream always points to the **active slot's backend port**. Switching traffic = updating the NGINX upstream config + `nginx -s reload`.

---

## Deployment Flow

### Step-by-Step: Deploying a New Release (Green)

Generate a Tech Spec that covers each of the following steps in full detail, including exact shell commands and GitHub Actions job definitions:

**1. Determine Active Slot**
- SSH into EC2
- Read the current active slot from a state file: `cat /opt/tms/active-slot` → outputs `blue` or `green`
- The new deployment target is always the **inactive slot**

**2. Pull New Docker Image (Backend only)**
- `docker pull <dockerhub-repo>/tms-backend:<new-tag>`
- Verify pull succeeded before proceeding
- *(Frontend is deployed automatically via AWS Amplify — no image pull required)*

**3. Inject Config & Secrets into Inactive Slot**
- Pull secrets from AWS Secrets Manager: `aws secretsmanager get-secret-value ...`
- Merge with `.env.<environment>` file
- Write merged config to the inactive slot's compose env file: `/opt/tms/green/.env`

**4. Start Green Container (Backend)**
- `docker compose -f /opt/tms/green/docker-compose.yml --env-file /opt/tms/green/.env up -d`
- Green backend starts on port `3002`
- Blue backend remains running and live on `3001` — **no downtime during this phase**
- *(Frontend Amplify deployment runs independently and is not part of this step)*

**5. Health Check Green Slot**
- Run health checks directly against Green's internal ports (not through NGINX)
- Do not switch traffic until all checks pass
- Health check specification (Backend only — Frontend health is managed by Amplify):

| Check | Target | Method | Expected | Retries | Interval | Timeout |
|---|---|---|---|---|---|---|
| Backend readiness | `http://localhost:3002/api/health` | GET | HTTP 200 | 5 | 10s | 5s |
| DB connectivity | `http://localhost:3002/api/health/db` | GET | HTTP 200 | 3 | 15s | 10s |

- If any health check fails after all retries → **abort deployment, stop Green container, Blue remains live**

**6. Approval Gate (PROD only)**
- GitHub Actions pipeline pauses here using `environment: production` protection rules
- Designated approvers review and approve in GitHub
- Only after approval does the pipeline proceed to traffic switch

**7. Switch Traffic: Blue → Green (NGINX Reload — Backend only)**
- Update NGINX upstream config to point to Green's backend port (`3002`)
- Run `nginx -t` to validate config syntax before applying
- Run `nginx -s reload` — zero-downtime reload, no dropped connections
- Write `green` to the state file: `echo "green" > /opt/tms/active-slot`
- Run a final external health check through NGINX to confirm Green is serving live traffic
- *(Frontend traffic switch is handled automatically by Amplify — no NGINX changes needed for UI)*

**NGINX upstream switch example (Backend):**
```nginx
# Before (Blue active)
upstream tms_backend {
    server 127.0.0.1:3001;
}

# After (Green active)
upstream tms_backend {
    server 127.0.0.1:3002;
}
```

**8. Post-Switch Validation Window**
- Monitor Green for **10 minutes** after cutover
- Pipeline polls health endpoints and checks error rate
- If failure detected within window → automatic rollback triggered (see Rollback Strategy)
- If window passes cleanly → proceed to Blue teardown

**9. Blue Slot Teardown (Backend)**
- After the 10-minute validation window with no rollback:
  - Stop and remove Blue container: `docker compose -f /opt/tms/blue/docker-compose.yml down`
  - Remove old Blue Docker images to free disk: `docker image prune -f`
- Blue's internal port (`3001`) is now free
- On the **next deployment**, Blue becomes the inactive slot and receives the new release
- *(No teardown needed for frontend — Amplify manages version history automatically)*

**10. Update Slot State**
- Blue slot is now free — next deploy will target Blue
- The cycle alternates: Blue → Green → Blue → Green on every deployment

---

## Directory Layout on EC2 (Backend only)

> **Frontend is on AWS Amplify** — no EC2 directory needed for UI.

Generate the Tech Spec to include this on-host directory structure:

```
/opt/tms/
├── active-slot              # contains "blue" or "green"
├── blue/
│   ├── docker-compose.yml   # Blue slot compose file (backend port 3001 only)
│   └── .env                 # Blue slot merged config (generated at deploy time)
├── green/
│   ├── docker-compose.yml   # Green slot compose file (backend port 3002 only)
│   └── .env                 # Green slot merged config (generated at deploy time)
└── nginx/
    ├── tms.conf             # NGINX config — backend upstream only (frontend served by Amplify)
    └── switch.sh            # Script to update backend upstream + reload NGINX
```

---

## GitHub Actions Pipeline Structure

Generate a GitHub Actions workflow YAML (`cd-pipeline.yml`) with jobs in this order:

**Backend (EC2 Blue-Green):**
```
prepare → pull-image → start-green → health-check → [approval: PROD only] → switch-traffic → validate-live → teardown-blue → notify
```

**Frontend (AWS Amplify):**
```
Amplify auto-triggers on branch push/merge → build (amplify.yml) → deploy to Amplify CDN → health check via Amplify Console
```
> GitHub Actions only needs to trigger Amplify manually if using `aws amplify start-job` for controlled deployments.

For each job specify:
- Job name and `needs` chain
- Runner: GitHub-hosted runner that SSHs into the EC2 instance via `appleboy/ssh-action` (confirm if self-hosted runner is preferred)
- Exact shell commands executed on the EC2 host
- Environment variables sourced from GitHub Secrets + AWS Secrets Manager
- Timeout per job
- On-failure behaviour (stop pipeline, keep Blue live, clean up Green)

---

## Notification & Observability

Specify notifications for each of these pipeline events:
- Deployment started (version, environment, target slot)
- Green health checks passed
- Approval requested (PROD only)
- Traffic switched to Green
- Blue teardown complete
- Deployment failed — Blue remains live (reason + pipeline link)
- Rollback triggered

Log aggregation target and deployment audit trail (GitHub Actions run URL) to be confirmed — see Gaps.

---

## Output Requirements

The generated Tech Spec must include:
- [ ] Single-machine Blue-Green architecture diagram (ASCII or Mermaid)
- [ ] Full `cd-pipeline.yml` GitHub Actions YAML
- [ ] NGINX upstream config before/after switch + `switch.sh` script
- [ ] On-host directory structure
- [ ] Health check script (bash, testing internal ports directly)
- [ ] Slot state management approach (`/opt/tms/active-slot` file)
- [ ] Docker Compose files for Blue and Green slots (with differing port mappings)
- [ ] Gaps section

---

## Gaps (Pre-filled — Confirm Before Generating)

| Gap | Assumption | Action Required |
|---|---|---|
| Traffic switch mechanism | NGINX on same EC2 host, port-based routing (backend only) | Confirm |
| Blue port | Backend: 3001 | Confirm or adjust |
| Green port | Backend: 3002 | Confirm or adjust |
| Frontend deployment | **AWS Amplify** — branch-connected, auto-build on merge | Confirm Amplify app is created and branches connected |
| Amplify custom domain | Assumed `tms.example.com` per env | Confirm domain and Amplify custom domain setup |
| Amplify environment variables | Non-sensitive in Amplify Console, secrets via AWS Secrets Manager | Confirm secret names and Amplify IAM role access |
| GitHub Actions runner | GitHub-hosted runner SSH-ing into EC2 (backend jobs only) | Confirm: GitHub-hosted vs self-hosted runner on EC2 |
| Database type | Not confirmed | **Required** — provide DB type (PostgreSQL / MySQL / MongoDB) |
| DB location | Not confirmed | Confirm — DB on same EC2 instance or separate RDS/instance |
| Monitoring/alerting stack | Not confirmed | Confirm (CloudWatch / Datadog / other) for error-rate polling in validation window |
| Notification channel | Not confirmed | Confirm (Slack / email / PagerDuty) |
| Post-switch validation window | Assumed 10 minutes | Confirm acceptable window |
| NGINX SSL/TLS termination | Not confirmed | Confirm if NGINX handles HTTPS or termination is upstream (e.g. ALB in front of EC2) |