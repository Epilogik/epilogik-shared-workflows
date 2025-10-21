# Epilogik Shared Workflows

> 🚀 Reusable GitHub Actions workflows and composite actions for deployment automation, semantic versioning, and release management.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-2088FF?logo=github-actions&logoColor=white)](https://github.com/features/actions)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [Quick Start](#-quick-start)
- [Reusable Workflows](#-reusable-workflows)
- [Composite Actions](#-composite-actions)
- [Examples](#-examples)
- [Best Practices](#-best-practices)
- [Contributing](#-contributing)

---

## 🎯 Overview

This repository provides battle-tested GitHub Actions workflows and composite actions that can be reused across multiple projects. It follows best practices for CI/CD, semantic versioning, and automated deployments.

### What's Included

- **Reusable Workflows**: Complete deployment and release workflows
- **Composite Actions**: Modular, reusable action components
- **Multi-language Support**: Works with .NET, Node.js, Python, Java, and more
- **Production-ready**: Used in production environments

---

## ✨ Features

### Workflows

- ✅ **Deploy to Hostinger**: SSH-based deployment with Docker Compose
- ✅ **Create Release PR**: Automated release branch and PR creation
- ✅ **Tag and Cleanup**: Automatic tagging and branch cleanup
- ✅ **Semantic Versioning**: Full semver support

### Actions

- 🚀 **SSH Deploy**: Deploy via SSH with Docker Compose support
- 🏷️ **Semantic Version**: Calculate next version automatically
- 📝 **Release Notes**: Generate beautiful release notes from commits

### General

- 🔒 **Security First**: Secrets management, SSH key validation
- 🎨 **Customizable**: Extensive configuration options
- 📊 **Observable**: Detailed logs and summaries
- ⚡ **Fast**: Optimized for performance

---

## 🚀 Quick Start

### 1. Use in Your Repository

Add this to your workflow file (`.github/workflows/deploy.yml`):

```yaml
name: Deploy

on:
  push:
    branches: [develop, main]

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: staging
      deploy_path: /home/deploy/myapp-staging
    secrets:
      ssh_host: ${{ secrets.STAGING_SSH_HOST }}
      ssh_user: ${{ secrets.STAGING_SSH_USER }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

### 2. Configure Secrets

Go to **Settings** → **Secrets and variables** → **Actions** and add:

```
SSH_PRIVATE_KEY=<your-ssh-private-key>
STAGING_SSH_HOST=your-server-ip
STAGING_SSH_USER=root
PROD_SSH_HOST=your-production-ip
PROD_SSH_USER=root
```

### 3. Create Environment

Go to **Settings** → **Environments** and create `staging` and `production`.

---

## 📦 Reusable Workflows

### `deploy-hostinger.yml`

Deploy applications to Hostinger VPS using SSH and Docker Compose.

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `environment` | ✅ | - | Environment name (staging/production) |
| `deploy_path` | ✅ | - | Deployment path on server |
| `compose_files` | ❌ | `docker-compose.infra.yml,docker-compose.services.yml` | Comma-separated list of compose files |
| `docker_network` | ❌ | `epilogik-net` | Docker network name |
| `wait_for_health` | ❌ | `true` | Wait for health checks |
| `cleanup_images` | ❌ | `true` | Cleanup old images |

**Secrets:**

| Name | Required | Description |
|------|----------|-------------|
| `ssh_host` | ✅ | SSH host address |
| `ssh_user` | ✅ | SSH username |
| `ssh_port` | ❌ | SSH port (default: 22) |
| `ssh_key` | ✅ | SSH private key |

**Example:**

```yaml
jobs:
  deploy:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: production
      deploy_path: /home/deploy/myapp
      compose_files: docker-compose.yml,docker-compose.db.yml
    secrets:
      ssh_host: ${{ secrets.PROD_SSH_HOST }}
      ssh_user: ${{ secrets.PROD_SSH_USER }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

### `create-release-pr.yml`

Create release PR with automatic version calculation and release notes.

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `base_branch` | ❌ | `develop` | Base branch for release |
| `bump_type` | ❌ | `patch` | Version bump (major/minor/patch or specific version) |
| `tag_prefix` | ❌ | `v` | Tag prefix |
| `auto_merge` | ❌ | `false` | Enable auto-merge |

**Outputs:**

| Name | Description |
|------|-------------|
| `release_version` | Calculated version |
| `release_branch` | Release branch name |
| `pr_number` | PR number |

**Example:**

```yaml
jobs:
  release:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/create-release-pr.yml@v1
    with:
      base_branch: develop
      bump_type: minor
```

---

### `tag-and-cleanup.yml`

Create tags and cleanup release branches after merge to main.

**Inputs:**

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `tag_prefix` | ❌ | `v` | Tag prefix |
| `delete_release_branch` | ❌ | `true` | Delete release branch |
| `create_github_release` | ❌ | `true` | Create GitHub Release |
| `main_branch` | ❌ | `main` | Main branch name |

**Example:**

```yaml
jobs:
  tag:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/tag-and-cleanup.yml@v1
    with:
      tag_prefix: v
      create_github_release: true
```

---

## 🧩 Composite Actions

### `actions/ssh-deploy`

Deploy via SSH with Docker Compose support.

**Usage:**

```yaml
- uses: Epilogik/epilogik-shared-workflows/actions/ssh-deploy@v1
  with:
    ssh_host: 185.139.1.15
    ssh_user: root
    ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
    deploy_path: /home/deploy/myapp
    environment: staging
    compose_files: docker-compose.yml
```

---

### `actions/semantic-version`

Calculate next semantic version.

**Usage:**

```yaml
- id: version
  uses: Epilogik/epilogik-shared-workflows/actions/semantic-version@v1
  with:
    bump_type: minor

- run: echo "Next version is ${{ steps.version.outputs.next_version }}"
```

---

### `actions/release-notes`

Generate release notes from commits.

**Usage:**

```yaml
- id: notes
  uses: Epilogik/epilogik-shared-workflows/actions/release-notes@v1
  with:
    format: markdown
    group_by: type

- run: echo "${{ steps.notes.outputs.release_notes }}"
```

---

## 📚 Examples

### Example 1: Complete .NET Deployment

```yaml
name: Deploy .NET App

on:
  push:
    branches: [develop, main]

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: staging
      deploy_path: /home/deploy/dotnet-app
      compose_files: docker-compose.yml
    secrets:
      ssh_host: ${{ secrets.STAGING_SSH_HOST }}
      ssh_user: ${{ secrets.STAGING_SSH_USER }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}

  create-release:
    if: github.ref == 'refs/heads/develop'
    needs: deploy-staging
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/create-release-pr.yml@v1
    with:
      bump_type: patch

  deploy-production:
    if: github.ref == 'refs/heads/main'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: production
      deploy_path: /home/deploy/dotnet-app
    secrets:
      ssh_host: ${{ secrets.PROD_SSH_HOST }}
      ssh_user: ${{ secrets.PROD_SSH_USER }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}

  tag-release:
    if: github.ref == 'refs/heads/main'
    needs: deploy-production
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/tag-and-cleanup.yml@v1
```

---

### Example 2: Node.js with Custom Version

```yaml
name: Release Node.js App

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Release version (e.g., 2.1.0)'
        required: true

jobs:
  create-release:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/create-release-pr.yml@v1
    with:
      bump_type: ${{ github.event.inputs.version }}
```

---

### Example 3: Multi-Environment Python App

```yaml
name: Deploy Python App

on:
  push:
    branches: [develop, staging, main]

jobs:
  deploy:
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: ${{ github.ref == 'refs/heads/main' && 'production' || github.ref_name }}
      deploy_path: /home/deploy/python-app
      compose_files: docker-compose.app.yml,docker-compose.db.yml
    secrets:
      ssh_host: ${{ secrets[format('{0}_SSH_HOST', github.ref == 'refs/heads/main' && 'PROD' || 'STAGING')] }}
      ssh_user: ${{ secrets[format('{0}_SSH_USER', github.ref == 'refs/heads/main' && 'PROD' || 'STAGING')] }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}
```

---

## 🎯 Best Practices

### 1. **Version Pinning**

Always pin to a specific version:

```yaml
uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
```

### 2. **Secret Management**

- Never hardcode secrets
- Use GitHub Environments for sensitive deployments
- Rotate SSH keys regularly

### 3. **Environment Setup**

Create environments with protection rules:
- **staging**: No approvers, auto-deploy
- **production**: Require approvals, manual deploy

### 4. **SSH Key Format**

Ensure your SSH key includes BEGIN/END markers:

```
-----BEGIN OPENSSH PRIVATE KEY-----
<key content>
-----END OPENSSH PRIVATE KEY-----
```

### 5. **Compose File Structure**

Organize by concern:
```
docker-compose.infra.yml    # Database, Redis, etc.
docker-compose.services.yml # Application services
```

---

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request

---

## 📄 License

MIT License - see [LICENSE](LICENSE) file for details

---

## 🆘 Support

- 📖 [Documentation](https://github.com/Epilogik/epilogik-shared-workflows/wiki)
- 🐛 [Issues](https://github.com/Epilogik/epilogik-shared-workflows/issues)
- 💬 [Discussions](https://github.com/Epilogik/epilogik-shared-workflows/discussions)

---

**Made with ❤️ by Epilogik Team**
