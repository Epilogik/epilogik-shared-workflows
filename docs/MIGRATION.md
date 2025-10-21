# Migration Guide

> How to migrate from local workflows to Epilogik Shared Workflows

---

## Overview

This guide helps you migrate your existing workflows to use the shared workflows repository.

### Benefits of Migration

✅ **Centralized Maintenance**: Update workflows in one place  
✅ **Consistency**: Same deployment logic across all projects  
✅ **Version Control**: Pin to specific versions (`@v1`, `@v2`)  
✅ **Reusability**: Use across .NET, Node.js, Python, etc.  
✅ **Less Code**: Reduce workflow file size by 70%+

---

## Before Migration

### Current Setup (Epilogik-Infra)

```
.github/workflows/
├── 1-auto-pr-to-develop.yml           (Keep as-is)
├── 2-deploy-to-hostinger.yml          → migrate
├── 3-create-release-pr.yml            → migrate
├── 4-deploy-production.yml            → migrate
├── 5-create-main-pr.yml               (Keep as-is)
├── 6-tag-and-cleanup.yml              → migrate
```

---

## Migration Steps

### Step 1: Update Workflow 2 (Deploy to Staging)

**Before** (`.github/workflows/2-deploy-to-hostinger.yml`):

```yaml
name: 2. Deploy to Staging (Hostinger)

on:
  push:
    branches:
      - develop

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          # ... 50+ lines of code ...
```

**After**:

```yaml
name: 2. Deploy to Staging

on:
  push:
    branches:
      - develop

jobs:
  deploy-staging:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: staging
      deploy_path: ${{ secrets.STAGING_DEPLOY_PATH }}
      compose_files: docker-compose.infra.yml,docker-compose.services.yml
      docker_network: epilogik-net
    secrets:
      ssh_host: ${{ secrets.STAGING_SSH_HOST }}
      ssh_user: ${{ secrets.STAGING_SSH_USER }}
      ssh_port: ${{ secrets.STAGING_SSH_PORT }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

**Lines of code**: 140 → 18 (87% reduction)

---

### Step 2: Update Workflow 3 (Create Release PR)

**Before**:

```yaml
name: 3. Create Release PR

on:
  workflow_run:
    workflows: ["2. Deploy to Staging (Hostinger)"]
    types: [completed]

jobs:
  create-release-pr:
    runs-on: ubuntu-latest
    steps:
      # ... 150+ lines of version calculation and PR creation ...
```

**After**:

```yaml
name: 3. Create Release PR

on:
  workflow_run:
    workflows: ["2. Deploy to Staging"]
    types: [completed]
    branches: [develop]

jobs:
  create-release-pr:
    if: github.event.workflow_run.conclusion == 'success'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/create-release-pr.yml@v1
    with:
      base_branch: develop
      bump_type: patch
```

**Lines of code**: 230 → 15 (93% reduction)

---

### Step 3: Update Workflow 4 (Deploy Production)

**Before**:

```yaml
name: 4. Deploy to Production

on:
  push:
    branches: ['release/**']

jobs:
  deploy-to-production:
    runs-on: ubuntu-latest
    environment: production
    steps:
      # ... similar 50+ lines of SSH deployment ...
```

**After**:

```yaml
name: 4. Deploy to Production

on:
  push:
    branches: ['release/**']

jobs:
  deploy-production:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: production
      deploy_path: ${{ secrets.PROD_DEPLOY_PATH }}
      compose_files: docker-compose.infra.yml,docker-compose.services.yml
      docker_network: epilogik-net
    secrets:
      ssh_host: ${{ secrets.PROD_SSH_HOST }}
      ssh_user: ${{ secrets.PROD_SSH_USER }}
      ssh_port: ${{ secrets.PROD_SSH_PORT }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

**Lines of code**: 140 → 18 (87% reduction)

---

### Step 4: Update Workflow 6 (Tag and Cleanup)

**Before**:

```yaml
name: 6. Tag Release and Cleanup

on:
  push:
    branches: [main]

jobs:
  create-tag-and-cleanup:
    runs-on: ubuntu-latest
    steps:
      # ... 100+ lines for tag creation and cleanup ...
```

**After**:

```yaml
name: 6. Tag and Cleanup

on:
  push:
    branches: [main]

jobs:
  tag-and-cleanup:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/tag-and-cleanup.yml@v1
    with:
      tag_prefix: v
      create_github_release: true
      delete_release_branch: true
```

**Lines of code**: 180 → 12 (93% reduction)

---

### Step 5: Keep Workflows 1 and 5

These workflows are project-specific and should remain as-is:

- `1-auto-pr-to-develop.yml` - Auto PR creation (custom logic)
- `5-create-main-pr.yml` - Main PR creation (triggers after workflow 4)

---

## Complete Migrated Setup

### Final `.github/workflows/` Structure

```
.github/workflows/
├── 1-auto-pr-to-develop.yml        (unchanged - 80 lines)
├── 2-deploy-to-staging.yml         (migrated - 18 lines)
├── 3-create-release-pr.yml         (migrated - 15 lines)
├── 4-deploy-production.yml         (migrated - 18 lines)
├── 5-create-main-pr.yml            (unchanged - 155 lines)
└── 6-tag-and-cleanup.yml           (migrated - 12 lines)
```

**Total lines before**: ~945 lines  
**Total lines after**: ~298 lines  
**Reduction**: 68% less code

---

## Testing the Migration

### 1. Test Staging Deployment

```bash
git checkout develop
git commit --allow-empty -m "test: trigger staging deployment"
git push origin develop
```

**Expected**: Workflow 2 runs → deploys to staging → Workflow 3 creates release PR

### 2. Test Production Deployment

1. Merge the release PR
2. **Expected**: Workflow 4 deploys to production → Workflow 5 creates main PR

### 3. Test Tag Creation

1. Merge the main PR
2. **Expected**: Workflow 6 creates tag and deletes release branch

---

## Rollback Plan

If something goes wrong, you can quickly rollback:

### Option 1: Pin to Previous Version

```yaml
uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v0.9
```

### Option 2: Revert to Local Workflows

```bash
git revert <migration-commit-hash>
git push origin develop
```

---

## Advanced: Custom Composite Actions

If you need custom deployment logic, you can create your own composite action:

### Example: Custom Deploy Action

```yaml
# .github/actions/custom-deploy/action.yml
name: 'Custom Deploy'
description: 'Custom deployment with extra steps'

inputs:
  environment:
    required: true

runs:
  using: 'composite'
  steps:
    # 1. Use shared SSH deploy
    - uses: Epilogik/epilogik-shared-workflows/actions/ssh-deploy@v1
      with:
        ssh_host: ${{ inputs.ssh_host }}
        # ... other inputs ...
    
    # 2. Add custom steps
    - shell: bash
      run: |
        echo "Running custom post-deployment tasks..."
        # Your custom logic here
```

### Usage

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: ./.github/actions/custom-deploy
        with:
          environment: staging
```

---

## Migration Checklist

- [ ] Clone `epilogik-shared-workflows` repository
- [ ] Update workflow 2 (staging deployment)
- [ ] Update workflow 3 (release PR)
- [ ] Update workflow 4 (production deployment)
- [ ] Update workflow 6 (tag and cleanup)
- [ ] Test staging deployment
- [ ] Test release PR creation
- [ ] Test production deployment
- [ ] Test tag creation
- [ ] Update README documentation
- [ ] Train team on new workflow
- [ ] Archive old workflow files (optional)

---

## Next Steps

1. **✅ Complete migration** following this guide
2. **📚 Read documentation** in epilogik-shared-workflows
3. **🔔 Watch** the shared-workflows repo for updates
4. **🤝 Contribute** improvements back to shared workflows
5. **📢 Share** with other teams

---

## Support

- **Issues**: [GitHub Issues](https://github.com/Epilogik/epilogik-shared-workflows/issues)
- **Discussions**: [GitHub Discussions](https://github.com/Epilogik/epilogik-shared-workflows/discussions)
- **Slack**: #devops channel

---

**Happy Deploying! 🚀**
