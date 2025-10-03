# Flutto CI/CD Workflows

Centralized repository for reusable GitHub Actions workflows used across all Flutto services (backend and frontend).

**Last Updated:** 2025-10-03

## Overview

This repository contains standardized CI/CD workflows that can be reused by all Flutto service repositories, ensuring consistent build, test, and deployment processes across the entire platform.

**Architecture:** Pull-based GitOps with ArgoCD Image Updater
- CI pushes Docker images to GHCR
- ArgoCD Image Updater polls GHCR every 2 minutes
- ArgoCD automatically syncs detected changes to Kubernetes

## Workflows

### 📦 Docker Build and Push (`docker-build-push.yml`)

Reusable workflow for building and pushing Docker images to GitHub Container Registry (GHCR).

**Features:**
- Multi-environment support (development, staging, production)
- Automatic tag generation based on branch (dev-{sha}, stage-{sha}, prod-{sha})
- BuildKit cache optimization with persistent cache mounts
- Kubernetes-compatible BuildKit network configuration
- Self-hosted runner support with Actions Runner Controller (ARC)
- **Note:** Manifest updates disabled - ArgoCD Image Updater handles deployment

**Usage:**
```yaml
jobs:
  build:
    uses: ZenVInnovations/cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: 'project_management_flutto'
      dockerfile_path: './docker/prod/Dockerfile'
      docker_target: 'slim'
      build_context: '.'
      fail_on_security: ''  # Optional: 'HIGH,CRITICAL' to fail on vulnerabilities
    secrets: inherit  # ← CRITICAL: Use inherit, not explicit GITHUB_TOKEN
```

### 🚀 Environment Promotion (`promote-environment.yml`)

**Status:** DISABLED - Use branch-based deployment instead

**Promotion Strategy:**
```bash
# Development → Staging
git checkout staging
git merge develop
git push origin staging

# Staging → Production
git checkout main
git merge staging
git push origin main
```

ArgoCD Image Updater automatically detects new images and updates deployments within 0-2 minutes.

### 🔒 Security Scanning (`security-scan.yml`)

Comprehensive security vulnerability scanning using Trivy scanner.

**Features:**
- Container image vulnerability scanning
- SARIF report generation saved as workflow artifacts (free alternative to GitHub Advanced Security)
- Configurable severity thresholds
- JSON and HTML reports for detailed analysis
- Non-blocking by default (set `fail_on_severity` to block on vulnerabilities)

**Usage:**
```yaml
jobs:
  security:
    needs: build
    uses: ZenVInnovations/cicd-workflows/.github/workflows/security-scan.yml@main
    with:
      image_name: 'project_management_flutto'
      image_tag: ${{ needs.build.outputs.image_tag }}
      fail_on_severity: ''  # Empty = non-blocking, 'HIGH,CRITICAL' = blocking
      upload_sarif: false  # True requires GitHub Advanced Security
    secrets: inherit
```

### 🔄 Emergency Rollback (`rollback.yml`)

**Status:** AVAILABLE - For emergency rollbacks via ArgoCD

**Recommended Rollback Method:**
```bash
# Option 1: ArgoCD rollback (recommended)
argocd app history flutto-dev-complete
argocd app rollback flutto-dev-complete <revision-id>

# Option 2: Kubernetes rollback
kubectl rollout undo deployment/openproject-dev -n flutto-dev

# Option 3: Manual image change
kubectl set image deployment/openproject-dev \
  openproject=ghcr.io/zenvinnovations/project_management_flutto:dev-<previous-sha> \
  -n flutto-dev
```

## Services Integration

### Backend (project_management_flutto)
- **Repository:** `ZenVInnovations/Project_Management_Flutto`
- **Container Registry:** `ghcr.io/zenvinnovations/project_management_flutto`
- **Environments:** development, staging, production, demo
- **URLs:**
  - Production: https://backend.flutto.ai (DISABLED)
  - Staging: https://stage-backend.flutto.ai
  - Development: https://develop-backend.flutto.ai
  - Demo: https://demo-backend.incube.cc

### Frontend (flutto-frontend)
- **Repository:** `ZenVInnovations/Flutto-Frontend`
- **Container Registry:** `ghcr.io/zenvinnovations/flutto-frontend`
- **Environments:** development, staging, production, demo
- **URLs:**
  - Production: https://flutto.ai (DISABLED)
  - Staging: https://staging.flutto.ai
  - Development: https://dev.flutto.ai
  - Demo: https://demo.incube.cc

## Environment Strategy

### Branch-to-Environment Mapping
- `main` → Production environment (`prod-{sha}` + `prod` + `latest`)
- `staging` → Staging environment (`stage-{sha}` + `stage`)
- `develop` → Development environment (`dev-{sha}` + `dev`)
- Feature branches → Feature environment (`feature-{branch}-{sha}`)
- Pull requests → PR environment (`pr-{number}-{sha}`)

### Tag Strategy (Current Convention)
```bash
# Development (develop branch)
ghcr.io/zenvinnovations/project_management_flutto:dev-8be6c53  # Specific SHA
ghcr.io/zenvinnovations/project_management_flutto:dev          # Rolling tag

# Staging (staging branch)
ghcr.io/zenvinnovations/project_management_flutto:stage-a3f92e1
ghcr.io/zenvinnovations/project_management_flutto:stage

# Production (main branch)
ghcr.io/zenvinnovations/project_management_flutto:prod-ff2fd1f
ghcr.io/zenvinnovations/project_management_flutto:prod
ghcr.io/zenvinnovations/project_management_flutto:latest
```

## Setup Requirements

### Repository Permissions (CRITICAL)

**DO NOT** set workflow-level `permissions:` block in calling workflows. This restricts permissions for reusable workflows.

**✅ CORRECT:**
```yaml
name: CI/CD Pipeline

# NO permissions block here - uses repository defaults

jobs:
  build:
    uses: ZenVInnovations/cicd-workflows/.github/workflows/docker-build-push.yml@main
    secrets: inherit  # ← Passes all secrets including GITHUB_TOKEN
```

**❌ WRONG:**
```yaml
name: CI/CD Pipeline

permissions:  # ← This RESTRICTS reusable workflows!
  contents: read
  packages: write

jobs:
  build:
    permissions: inherit  # ← INVALID SYNTAX (only secrets: inherit exists)
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # ← NOT DEFINED in reusable workflow
```

### Repository Secrets

No secrets required! `GITHUB_TOKEN` is automatically available with repository default permissions.

**Optional Secrets:**
- `MANIFEST_UPDATE_TOKEN`: Only if enabling manifest updates (currently disabled)

### Service Configuration

**Backend Example** (`Project_Management_Flutto/.github/workflows/ci-cd.yml`):
```yaml
name: Backend CI/CD Pipeline

on:
  push:
    branches: [main, staging, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main, staging]

env:
  SERVICE_NAME: project_management_flutto
  RUBY_VERSION: '3.4.2'

jobs:
  test:
    name: Backend Health Check
    runs-on: ubuntu-latest
    # ... health check steps ...

  build:
    needs: test
    uses: ZenVInnovations/cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: 'project_management_flutto'
      dockerfile_path: './docker/prod/Dockerfile'
      docker_target: 'slim'
      build_context: '.'
      fail_on_security: ''
    secrets: inherit

  security:
    needs: build
    if: always() && needs.build.result == 'success'
    uses: ZenVInnovations/cicd-workflows/.github/workflows/security-scan.yml@main
    with:
      image_name: 'project_management_flutto'
      image_tag: ${{ needs.build.outputs.image_tag }}
      fail_on_severity: ''
      upload_sarif: false
    secrets: inherit
```

**Frontend Example** (`Flutto-Frontend/.github/workflows/ci-cd.yml`):
```yaml
jobs:
  build:
    uses: ZenVInnovations/cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: 'flutto-frontend'
      dockerfile_path: './Dockerfile'
      build_context: '.'
    secrets: inherit
```

## ArgoCD Image Updater Integration

These workflows implement **pull-based GitOps** with ArgoCD Image Updater:

### Deployment Flow

1. **CI Push to GHCR** (6-8 minutes)
   ```
   Push to branch → GitHub Actions runs → Docker build → Push to GHCR
   ```

2. **Image Updater Detection** (0-2 minutes)
   ```
   ArgoCD Image Updater polls GHCR every 2 minutes → Detects new digest
   ```

3. **Application Update** (~1 second)
   ```
   Image Updater updates Application parameters using write-back-method: argocd
   ```

4. **ArgoCD Sync** (3-5 minutes)
   ```
   ArgoCD detects parameter change → Syncs to Kubernetes → Rolling update
   ```

**Total Time:** 10-15 minutes from push to live

### Image Updater Configuration

ArgoCD Applications are configured with annotations:
```yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: >
    backend=ghcr.io/zenvinnovations/project_management_flutto
  argocd-image-updater.argoproj.io/backend.update-strategy: newest-build
  argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^dev.*$
  argocd-image-updater.argoproj.io/write-back-method: argocd
```

**Key Point:** CI does NOT update Git manifests. That step is disabled with `if: false` in the workflow.

## BuildKit Configuration for Kubernetes

**Critical for self-hosted runners in Kubernetes (Actions Runner Controller):**

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
  with:
    driver: docker-container
    driver-opts: network=host  # ← REQUIRED for K8s networking
    buildkitd-flags: --allow-insecure-entitlement network.host

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    allow: network.host  # ← Enable network access during build
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

**Why This is Required:**
- BuildKit containers in K8s have isolated network namespaces
- Default configuration cannot resolve DNS or access network
- `network=host` shares runner pod's network namespace
- Fixes: `fatal: unable to access 'https://github.com/...'` errors

## Performance Metrics

### CI/CD Timings (Current)

**Backend (Project_Management_Flutto):**
- Health checks: ~3 min
- Docker build (cached): ~3-5 min
- Docker build (first time): ~10-12 min
- Security scan: ~2 min
- **Total CI time:** ~6-8 min (85% faster than original 40+ min)

**Build Optimizations Applied:**
- BIM support disabled: 18 min → 3 min (85% reduction)
- BuildKit cache mounts: 60% faster
- Multi-stage build with slim target
- Ruby 3.4.2 + Rails 8 compatibility fixes

**Frontend (Flutto-Frontend):**
- Build and push: ~4-6 min
- Security scan: ~1-2 min
- **Total CI time:** ~5-8 min

### End-to-End Deployment Time
- CI (build + push): 6-8 min
- Image Updater detection: 0-2 min
- ArgoCD sync: 3-5 min
- Pod startup: 2-3 min
- **Total:** 10-15 minutes from push to live

## Security Features

- **Image Scanning:** Trivy vulnerability scanning with configurable severity gates
- **SARIF Reports:** Saved as workflow artifacts (free alternative to GitHub Advanced Security)
- **Secret Management:** Automatic `GITHUB_TOKEN` with repository permissions
- **Non-Blocking Scans:** Security scans don't fail builds by default
- **Container Registry:** GHCR with private image support

## Emergency Procedures

### Quick Rollback

**ArgoCD Rollback (Recommended):**
```bash
# View deployment history
argocd app history flutto-dev-complete

# Rollback to previous version
argocd app rollback flutto-dev-complete <revision-id>
```

**Kubernetes Rollback:**
```bash
# Rollback deployment
kubectl rollout undo deployment/openproject-dev -n flutto-dev

# Rollback to specific revision
kubectl rollout undo deployment/openproject-dev -n flutto-dev --to-revision=3
```

**Manual Image Change:**
```bash
# Set specific image version
kubectl set image deployment/openproject-dev \
  openproject=ghcr.io/zenvinnovations/project_management_flutto:dev-8be6c53 \
  -n flutto-dev
```

### Stop Auto-Deployment

```bash
# Disable ArgoCD auto-sync
argocd app set flutto-dev-complete --sync-policy none

# Disable Image Updater (scale to 0)
kubectl scale deployment argocd-image-updater -n argocd --replicas=0

# Re-enable later
kubectl scale deployment argocd-image-updater -n argocd --replicas=1
argocd app set flutto-dev-complete --sync-policy automated
```

## Troubleshooting

### Build Fails with "permission_denied: write_package"

**Cause:** Workflow-level `permissions:` block restricts reusable workflows

**Fix:**
1. Remove entire workflow-level `permissions:` block
2. Use `secrets: inherit` instead of explicit `GITHUB_TOKEN`
3. Repository defaults (write for all scopes) will apply

### Build Fails with Network Errors in K8s

**Cause:** BuildKit network isolation in Kubernetes

**Fix:** Add to centralized workflow:
```yaml
driver-opts: network=host
allow: network.host
```

### Image Updater Not Detecting New Images

**Cause:** Polling interval or tag mismatch

**Fix:**
```bash
# Force Image Updater to poll immediately
kubectl rollout restart deployment argocd-image-updater -n argocd

# Check Image Updater logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater --tail=100 -f

# Verify tag pattern matches
kubectl get application flutto-dev-complete -n argocd -o yaml | grep allow-tags
```

## Monitoring & Observability

- **GitHub Actions:** Workflow run history and logs
- **ArgoCD Dashboard:** Real-time deployment status and sync health
- **Kubernetes:** Pod status, logs, and events via `kubectl`
- **GHCR:** Container image registry with vulnerability scanning
- **Application Health:** Automated health checks post-deployment

## Documentation

**Backend:**
- [CI/CD Complete Guide](/home/Project_Management_Flutto/CI_CD_COMPLETE_GUIDE.md)
- [CD GitOps Architecture](/home/Project_Management_Flutto/CD_GITOPS_ARCHITECTURE.md)
- [Quick CI/CD Reference](/home/Project_Management_Flutto/QUICK_CICD_REFERENCE.md)

**GitOps:**
- [CD Complete Guide](/home/flutto-gitops/docs/CD-COMPLETE-GUIDE.md)
- [ApplicationSets](/home/flutto-gitops/applicationsets/hub/)

## Contributing

1. Test changes in a non-production environment first
2. Update this README with any workflow changes
3. Follow existing patterns for consistency
4. Document any new features or breaking changes
5. Get approval before merging to `main`

## Support

**Quick Links:**
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [ArgoCD Image Updater Docs](https://argocd-image-updater.readthedocs.io/)
- [BuildKit Documentation](https://docs.docker.com/build/buildkit/)

**For Issues:**
- Create an issue in the relevant repository
- Check documentation guides first
- Include workflow run IDs and error messages