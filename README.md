# Flutto CI/CD Workflows

Centralized repository for reusable GitHub Actions workflows used across all Flutto services.

## Overview

This repository contains standardized CI/CD workflows that can be reused by all Flutto service repositories, ensuring consistent build, test, and deployment processes across the entire platform.

## Workflows

### 📦 Docker Build and Push (`docker-build-push.yml`)

Reusable workflow for building and pushing Docker images to GitHub Container Registry (GHCR).

**Features:**
- Multi-environment support (development, staging, production)
- Automatic tag generation based on branch
- Manifest file updates for GitOps deployment
- Build context and target customization

**Usage:**
```yaml
jobs:
  build:
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: 'your-service-name'
      dockerfile_path: './Dockerfile'
      build_context: '.'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      MANIFEST_UPDATE_TOKEN: ${{ secrets.MANIFEST_UPDATE_TOKEN }}
```

### 🚀 Environment Promotion (`promote-environment.yml`)

Automated workflow for promoting deployments between environments with proper review processes.

**Features:**
- Multi-environment promotion (dev → staging → production)
- Automated changelog generation
- Pull request creation with deployment checklists
- Environment-specific reviewer assignment

**Usage:**
```yaml
jobs:
  promote:
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/promote-environment.yml@main
    with:
      service_name: 'your-service-name'
      from_environment: 'development'
      to_environment: 'staging'
      from_branch: 'develop'
      to_branch: 'staging'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 🔒 Security Scanning (`security-scan.yml`)

Comprehensive security vulnerability scanning using Trivy scanner.

**Features:**
- Container image vulnerability scanning
- SARIF report generation for GitHub Security
- Configurable severity thresholds
- Pull request comments with scan results

**Usage:**
```yaml
jobs:
  security:
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/security-scan.yml@main
    with:
      image_name: 'your-service-name'
      image_tag: 'your-image-tag'
      fail_on_severity: 'HIGH,CRITICAL'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### 🔄 Emergency Rollback (`rollback.yml`)

Emergency rollback system with manual and comment-triggered execution.

**Features:**
- Manual workflow dispatch for emergency rollbacks
- Comment-triggered rollbacks (`/rollback service environment [version]`)
- Automatic version detection from git history
- Image verification before rollback

**Usage:**

**Manual Trigger:**
- Go to Actions tab → Emergency Rollback → Run workflow

**Comment Trigger:**
```
/rollback flutto-frontend production v20240126.143022
/rollback project_management_flutto staging
```

## Services Integration

### Frontend (flutto-frontend)
- **Repository:** `zenvinnovations/flutto-frontend`
- **Environments:** development, staging, production
- **URLs:**
  - Production: https://zenv.flutto.ai
  - Staging: https://staging.flutto.ai
  - Development: https://dev.flutto.ai

### Backend (project_management_flutto)
- **Repository:** `zenvinnovations/project_management_flutto`
- **Environments:** development, staging, production
- **URLs:**
  - Production: https://backend.flutto.ai
  - Staging: https://stage-backend.flutto.ai
  - Development: https://develop-backend.flutto.ai

## Environment Strategy

### Branch-to-Environment Mapping
- `main` → Production environment (`v{timestamp}`)
- `staging` → Staging environment (`staging-v{timestamp}-rc.{build}`)
- `develop` → Development environment (`dev-{short_sha}`)
- Feature branches → Feature environment (`feature-{branch}-{short_sha}`)
- Pull requests → PR environment (`pr-{number}-{short_sha}`)

### Tag Strategy
- **Production:** `v20240126.143022` (v + timestamp)
- **Staging:** `staging-v20240126.143022-rc.123` (with release candidate)
- **Development:** `dev-a1b2c3d` (dev + short SHA)

## Setup Requirements

### Repository Secrets

Each repository using these workflows needs the following secrets:

```yaml
GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}  # Automatic
MANIFEST_UPDATE_TOKEN: # Personal access token for manifest updates
```

### Service Configuration

Update your service repository's workflow file:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, staging, develop]
  pull_request:
    branches: [main, staging]

jobs:
  build:
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: 'your-service-name'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      MANIFEST_UPDATE_TOKEN: ${{ secrets.MANIFEST_UPDATE_TOKEN }}

  security:
    needs: build
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/security-scan.yml@main
    with:
      image_name: 'your-service-name'
      image_tag: ${{ needs.build.outputs.image_tag }}
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## ArgoCD Integration

These workflows are designed to work with ArgoCD GitOps deployment:

1. **Build & Push:** Workflows build images and push to GHCR
2. **Manifest Update:** Automatically update deployment manifests in `flutto-deploy-manifests` repository
3. **ArgoCD Sync:** ArgoCD detects changes and syncs deployments automatically
4. **Monitoring:** Use rollback workflows if issues are detected

## Security Features

- **Image Scanning:** Trivy vulnerability scanning with severity gates
- **SARIF Reports:** Integration with GitHub Security tab
- **Secret Management:** Proper secret handling with GitHub Secrets
- **Access Control:** Environment-specific reviewer requirements

## Emergency Procedures

### Quick Rollback
```bash
# Comment on any issue/PR:
/rollback flutto-frontend production

# Or use GitHub UI:
Actions → Emergency Rollback → Run workflow
```

### Manual Intervention
```bash
# Check current deployment
kubectl get pods -n flutto-frontend-production

# Manual rollback via kubectl
kubectl rollout undo deployment/flutto-frontend -n flutto-frontend-production

# Check ArgoCD status
argocd app get flutto-frontend-production
```

## Monitoring & Alerts

- **GitHub Actions:** All workflow runs are logged and monitored
- **ArgoCD Dashboard:** Real-time deployment status
- **Netdata:** Infrastructure monitoring with alerts
- **Application Health:** Automated health checks post-deployment

## Contributing

1. Fork this repository
2. Create a feature branch
3. Test changes with a service repository
4. Submit a pull request with detailed description
5. Get approval from @srimanreddy99

## Support

For issues or questions:
- Create an issue in this repository
- Tag @srimanreddy99 for urgent matters
- Check existing workflows for examples