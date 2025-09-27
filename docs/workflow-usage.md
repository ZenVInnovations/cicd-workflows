# Workflow Usage Guide

This document provides detailed examples and best practices for using Flutto CI/CD workflows.

## Complete Service Integration Example

### Frontend Service (flutto-frontend)

Create `.github/workflows/ci-cd.yml` in your frontend repository:

```yaml
name: Frontend CI/CD Pipeline

on:
  push:
    branches: [main, staging, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main, staging]

env:
  SERVICE_NAME: flutto-frontend

jobs:
  # Build and Push Docker Image
  build:
    name: 🐳 Build & Push
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: ${{ env.SERVICE_NAME }}
      dockerfile_path: './Dockerfile'
      build_context: '.'
      build_script: './scripts/build-frontend.sh'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      MANIFEST_UPDATE_TOKEN: ${{ secrets.MANIFEST_UPDATE_TOKEN }}

  # Security Vulnerability Scanning
  security:
    name: 🔒 Security Scan
    needs: build
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/security-scan.yml@main
    with:
      image_name: ${{ env.SERVICE_NAME }}
      image_tag: ${{ needs.build.outputs.image_tag }}
      fail_on_severity: 'HIGH,CRITICAL'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Auto-promote to next environment
  promote-to-staging:
    name: 🚀 Promote to Staging
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    needs: [build, security]
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/promote-environment.yml@main
    with:
      service_name: ${{ env.SERVICE_NAME }}
      from_environment: 'development'
      to_environment: 'staging'
      from_branch: 'develop'
      to_branch: 'staging'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  promote-to-production:
    name: 🚀 Promote to Production
    if: github.ref == 'refs/heads/staging' && github.event_name == 'push'
    needs: [build, security]
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/promote-environment.yml@main
    with:
      service_name: ${{ env.SERVICE_NAME }}
      from_environment: 'staging'
      to_environment: 'production'
      from_branch: 'staging'
      to_branch: 'main'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Backend Service (project_management_flutto)

Create `.github/workflows/ci-cd.yml` in your backend repository:

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

jobs:
  # Run Tests First
  test:
    name: 🧪 Run Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.1
          bundler-cache: true
      - name: Run tests
        run: |
          bundle exec rspec
          bundle exec rubocop

  # Build and Push Docker Image
  build:
    name: 🐳 Build & Push
    needs: test
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/docker-build-push.yml@main
    with:
      service_name: ${{ env.SERVICE_NAME }}
      dockerfile_path: './Dockerfile'
      docker_target: 'slim'
      build_context: '.'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      MANIFEST_UPDATE_TOKEN: ${{ secrets.MANIFEST_UPDATE_TOKEN }}

  # Security Vulnerability Scanning
  security:
    name: 🔒 Security Scan
    needs: build
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/security-scan.yml@main
    with:
      image_name: ${{ env.SERVICE_NAME }}
      image_tag: ${{ needs.build.outputs.image_tag }}
      fail_on_severity: 'CRITICAL'  # Only fail on critical for backend
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Environment Promotions
  promote-to-staging:
    name: 🚀 Promote to Staging
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    needs: [build, security]
    uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/promote-environment.yml@main
    with:
      service_name: ${{ env.SERVICE_NAME }}
      from_environment: 'development'
      to_environment: 'staging'
      from_branch: 'develop'
      to_branch: 'staging'
    secrets:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Rollback Examples

### Emergency Rollback via Comments

Add comments to any issue or pull request:

```bash
# Rollback frontend to previous version
/rollback flutto-frontend production

# Rollback backend to specific version
/rollback project_management_flutto staging v20240125.120000

# Using service aliases
/rollback frontend production
/rollback backend staging dev-a1b2c3d
```

### Manual Rollback Workflow

1. Go to Actions tab in this repository
2. Select "Emergency Rollback" workflow
3. Click "Run workflow"
4. Fill in parameters:
   - Service: `flutto-frontend` or `project_management_flutto`
   - Environment: `development`, `staging`, or `production`
   - Target Version: (optional - leave empty for previous version)
   - Reason: Brief description of why rollback is needed

## Workflow Triggers

### Automatic Triggers

```yaml
# Trigger on push to main branches
on:
  push:
    branches: [main, staging, develop]

# Trigger on pull requests
on:
  pull_request:
    branches: [main, staging]

# Trigger on specific paths
on:
  push:
    paths:
      - 'src/**'
      - 'Dockerfile'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

### Manual Triggers

```yaml
# Allow manual workflow execution
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - development
          - staging
          - production
```

## Environment-Specific Configuration

### Development Environment
- **Auto-deploy:** Yes (from develop branch)
- **Security gates:** Warn only
- **Rollback:** Available but not critical

### Staging Environment
- **Auto-deploy:** Via promotion PR
- **Security gates:** Block on HIGH+CRITICAL
- **Rollback:** Available with notification
- **Required approvals:** 1

### Production Environment
- **Auto-deploy:** Via promotion PR only
- **Security gates:** Block on CRITICAL only
- **Rollback:** Immediate availability
- **Required approvals:** 2
- **Additional checks:** Health monitoring, canary deployment

## Troubleshooting

### Common Issues

**1. Manifest Update Failed**
```yaml
Error: Failed to update tag in manifest
```
Solution: Check MANIFEST_UPDATE_TOKEN permissions in repository secrets.

**2. Security Scan Blocking Deployment**
```yaml
Error: Security gate failed: Critical vulnerabilities found
```
Solution: Update base images and dependencies, then re-run the workflow.

**3. Image Not Found During Rollback**
```yaml
Error: Image does not exist in GHCR
```
Solution: Verify the target version exists in GitHub Container Registry.

### Debug Steps

1. **Check workflow logs:** Actions tab → Failed workflow → View logs
2. **Verify secrets:** Repository Settings → Secrets and variables
3. **Test image build:** Run docker build locally
4. **Check manifest repository:** Verify flutto-deploy-manifests has correct permissions

## Best Practices

### Workflow Organization
- Keep workflows simple and focused
- Use clear naming conventions
- Add meaningful descriptions to all inputs
- Include proper error handling

### Security
- Use GitHub Secrets for all sensitive data
- Limit workflow permissions to minimum required
- Regularly update workflow dependencies
- Monitor security scan results

### Performance
- Use workflow caching where possible
- Minimize external dependencies
- Parallel job execution when safe
- Clean up artifacts regularly

## Advanced Usage

### Custom Build Scripts

```yaml
# Frontend with custom build
uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/docker-build-push.yml@main
with:
  service_name: 'flutto-frontend'
  build_script: './scripts/build-prod.sh'
```

### Multi-target Docker Builds

```yaml
# Backend with specific target
uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/docker-build-push.yml@main
with:
  service_name: 'project_management_flutto'
  docker_target: 'production'
```

### Conditional Security Scanning

```yaml
# Skip security scan for development
security:
  if: github.ref != 'refs/heads/develop'
  uses: zenvinnovations/flutto-cicd-workflows/.github/workflows/security-scan.yml@main
```