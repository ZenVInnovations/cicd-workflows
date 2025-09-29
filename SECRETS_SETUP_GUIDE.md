# GitHub Actions Secrets Setup Guide

## Issue Fixed

Resolved authentication issues where self-hosted runners couldn't pull Docker images from GitHub Container Registry (GHCR).

**Error was:**
```
Error response from daemon: denied
```

## Root Cause

The centralized workflows (`security-scan.yml` and `docker-build-push.yml`) weren't declaring `GITHUB_TOKEN` as required secrets for `workflow_call`, causing authentication failures on self-hosted runners.

## Solution Applied

### 1. Fixed Workflow Declarations

**Added GHCR_PAT to reusable workflows:**
- `.github/workflows/security-scan.yml`
- `.github/workflows/docker-build-push.yml`

**Changed from:**
```yaml
on:
  workflow_call:
    inputs:
      # ... inputs ...
    # Missing authentication token declaration
```

**To:**
```yaml
on:
  workflow_call:
    inputs:
      # ... inputs ...
    secrets:
      GHCR_PAT:
        required: true
      MANIFEST_UPDATE_TOKEN:
        required: true
```

### 2. Removed Duplicate Workflows

Deleted the `workflows/` directory which contained outdated duplicates of the workflows with different configurations.

## Required Repository Secrets

Set these up in **GitHub Repository Settings → Secrets and Variables → Actions**:

### 1. MANIFEST_UPDATE_TOKEN (Required)

**Purpose:** Updates deployment manifests in the flutto-deployment-manifests repository

**How to create:**
1. Go to GitHub → Settings → Developer settings → Personal access tokens
2. Generate new token (classic)
3. Required scopes:
   - ✅ `repo` (Full control of private repositories)
   - ✅ `workflow` (Update GitHub Action workflows)
4. Copy the token and add to repository secrets as `MANIFEST_UPDATE_TOKEN`

### 2. GHCR_PAT (Required)

**Purpose:** Authenticates Docker pulls and pushes to GitHub Container Registry (GHCR)

**How to create:**
1. Go to GitHub → Settings → Developer settings → Personal access tokens
2. Generate new token (classic)
3. Required scopes:
   - ✅ `read:packages` (Pull Docker images from GHCR)
   - ✅ `write:packages` (Push Docker images to GHCR)
4. Copy the token and add to repository secrets as `GHCR_PAT`

**Note:** This provides dedicated authentication for container registry operations, separate from general GitHub API access.

## Self-Hosted Runner Requirements

### Runner Registration PAT Scopes

The Personal Access Token used to register your self-hosted runners must have:

```bash
Required Scopes:
✅ repo                 # Full control of private repositories
✅ admin:org            # Organization runner management
✅ workflow             # Update GitHub Action workflows
✅ read:packages        # Pull Docker images from GHCR
✅ write:packages       # Push Docker images to GHCR
```

### Verify Runner Authentication

Check if your runners can authenticate:

```bash
# Check runner status
kubectl get pods -n arc-runners

# Check if runners can pull images
kubectl exec -it <runner-pod> -n arc-runners -- docker pull ghcr.io/zenvinnovations/flutto-frontend:latest
```

## Testing the Fix

### 1. Trigger a New Workflow

Push a change to the frontend repository to trigger the CI/CD pipeline:

```bash
cd /home/Flutto-Frontend
# Make a small change
echo "# Test authentication fix" >> test-auth-fix.md
git add test-auth-fix.md
git commit -m "test: verify authentication fix for self-hosted runners"
git push origin develop
```

### 2. Expected Results

The workflow should now:
- ✅ **Build step**: Successfully authenticate and push to GHCR
- ✅ **Security scan step**: Successfully authenticate and pull from GHCR
- ✅ **No "denied" errors**: Proper authentication throughout

### 3. Monitor Progress

Watch the workflow at: https://github.com/ZenVInnovations/Flutto-Frontend/actions

## Troubleshooting

### If authentication still fails:

1. **Check PAT scopes** for the runner registration token
2. **Verify secrets** are properly set in repository settings
3. **Check runner logs**:
   ```bash
   kubectl logs -n arc-runners <runner-pod-name>
   ```

### If workflows still use old versions:

GitHub Actions may cache workflow references. Wait a few minutes or manually re-run the workflow.

## Security Notes

- **Rotate PATs regularly** (recommend 90-day expiration)
- **Use GitHub Apps** instead of PATs for production (better security and rate limits)
- **Monitor runner logs** for authentication issues
- **Keep runner software updated** for latest security features

## Files Changed

- `.github/workflows/security-scan.yml` - Added GITHUB_TOKEN declaration
- `.github/workflows/docker-build-push.yml` - Added GITHUB_TOKEN declaration
- Removed `workflows/` directory - Eliminated conflicting duplicates
- Added this documentation file