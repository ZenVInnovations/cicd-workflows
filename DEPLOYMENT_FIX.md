# Deployment Manifest Fix

## Issue Resolved

Fixed the CI/CD workflow error: "⚠️ Manifest not found: argocd/flutto-frontend-development.yaml"

## Root Cause

The deployment architecture was modernized to use:
- **ArgoCD ApplicationSets** for automatic application generation
- **Helm charts** with environment-specific values
- **Unified platform deployment** instead of individual service manifests

However, the CI/CD workflow was still looking for individual ArgoCD application manifests that no longer existed.

## Solution Applied

### Updated CI/CD Workflows

Modified both workflow files:
- `cicd-workflows/.github/workflows/docker-build-push.yml`
- `cicd-workflows/workflows/docker-build-push.yml`

**Key Changes:**
1. **Repository URL**: Changed from `flutto-deploy-manifests` to `flutto-deployment-manifests`
2. **Target Files**: Now updates Helm values files instead of individual manifests
3. **Update Logic**: Uses `yq` to update `{service}.image.tag` in values files
4. **Environment Mapping**: Maps `development` → `dev` for values file names

### Environment and Service Mapping

| Environment | Values File | Frontend Key | Backend Key |
|------------|------------|--------------|-------------|
| development | `platform/helm/flutto-platform/values/dev.yaml` | `frontend.image.tag` | `openproject.image.tag` |
| staging | `platform/helm/flutto-platform/values/staging.yaml` | `frontend.image.tag` | `openproject.image.tag` |
| production | `platform/helm/flutto-platform/values/production.yaml` | `frontend.image.tag` | `openproject.image.tag` |

## New Deployment Flow

1. **Build Phase**: Creates Docker image with appropriate tag
2. **Manifest Update**: Updates Helm values file with new image tag
3. **ArgoCD Sync**: ApplicationSet automatically detects changes and deploys

## Testing the Fix

To test the updated workflow:

```bash
# Push to develop branch (triggers development deployment)
git push origin develop

# Push to staging branch (triggers staging deployment)
git push origin staging

# Push to main branch (triggers production deployment)
git push origin main
```

## Verification

After deployment, verify the changes in the manifests repository:
- Check the updated values file has the correct image tag
- Confirm ArgoCD has synced the changes
- Validate the application is running with the new image

## Benefits

1. **Compatibility**: Works with modern Helm-based deployment structure
2. **Reliability**: Uses `yq` for precise YAML updates
3. **Visibility**: Better logging and error reporting
4. **Maintainability**: Single source of truth for environment configuration