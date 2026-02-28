## Production Readiness Checklist (DevOps)
Before every PROD deployment, verify the following:

### 1. Build & Docker Image
- [ ] Docker image builds successfully
- [ ] Image tagged with version + git SHA (e.g. 1.2.0-abc1234)
- [ ] Image pushed to Docker Hub
- [ ] No hardcoded secrets inside the image (Snyk / Trivy scan)

### 2. CI/CD Pipeline
- [ ] All GitHub Actions stages passed (build, test, scan, publish)
- [ ] Jest unit tests pass with 0 failures
- [ ] Snyk scan — no critical CVEs
- [ ] Manual approval gate triggered for PROD

### 3. Configuration & Secrets
- [ ] All secrets stored in AWS Secrets Manager
- [ ] PROD .env file reviewed — only non-sensitive config values
- [ ] No secrets found in Git history

### 4. Rollback Readiness
- [ ] Previous working Docker image available in Docker Hub
- [ ] Rollback runbook documented and reviewed
- [ ] Rollback triggers defined and communicated to the team