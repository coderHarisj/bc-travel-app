---
trigger: always_on
---

---
name: devops-context
description: Global DevOps context injected into every AI interaction
alwaysApply: true
---

# Project DevOps Context

## Application
- Name: Transport Management System (TMS)
- Stack: Frontend — React (Node 24) | Backend — Node.js + Express.js (Node 24)
- Repo: GitHub
- test stack: jest
- security scan: synk
- Registry: Docker Hub
- Platform: AWS: Backend -Direct server deployment (no orchestration platform), Frontend - Amplify for deployment

## Environments
- Branches: dev/* → DEV | test→ QA | uat/* → UAT | prod → PROD
- Config tool: Environment variables via .env files per environment
- Secrets tool: AWS Secrets Manager
- Config Management: both Config (.env) + AWS secret manager works together for deployment

## Standards
- All pipeline code: GitHub Actions YAML
- All specs: structured Markdown with tables
- Versioning: semantic (MAJOR.MINOR.PATCH), conventional commits
- Rollback: always include step-by-step runbook + DB migration table
- Output: flag missing context in a "Gaps" section at the end

## CI/CD Pipeline Behavior
- On push to dev/*  → build + test + lint + security scan → deploy to DEV
- On merge to test/*   → build + test + lint + security scan → deploy to QA
- On merge & tag creation to uat/*  → build + test + security scan → deploy to UAT
- On merge & tag creation to prod → build + test + scan + approval gate → deploy to PROD
- All pipelines use GitHub Actions YAML
- Docker image tagged with semantic version + git SHA
- Images pushed to Docker Hub after successful build

## Rollback Standard
- Every deployment must include a rollback runbook
- DB migrations must include a rollback column in a table format:

| Migration | Up | Down | Risk |
|---|---|---|---|
| example | ALTER TABLE ADD COLUMN | ALTER TABLE DROP COLUMN | Low |

## Deployment Strategy
- Blue-Green
- Zero Downtime

## Rollback Strategy


## Gaps
- Deployment target OS not confirmed — assuming Ubuntu 22.04 LTS + PM2 as process manager. Confirm if different.
- Database type not provided — confirm (e.g. PostgreSQL, MySQL, MongoDB) to complete migration and rollback standards.
- Docker Compose usage not confirmed — confirm if Docker Compose is used for local dev and/or server deployment.