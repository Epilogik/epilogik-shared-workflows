# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-10-21

### Added

#### Reusable Workflows
- **deploy-hostinger.yml**: Complete SSH deployment workflow with Docker Compose support
- **create-release-pr.yml**: Automated release PR creation with semantic versioning
- **tag-and-cleanup.yml**: Automatic tagging and release branch cleanup

#### Composite Actions
- **ssh-deploy**: Modular SSH deployment action with Docker Compose
  - SSH key validation and setup
  - Multi-file deployment support
  - Health check waiting
  - Image cleanup
  - Detailed logging

- **semantic-version**: Smart version calculation
  - Auto-detect latest tag
  - Support for major/minor/patch bumps
  - Custom version support
  - Multiple outputs (version, tag, branch)

- **release-notes**: Automated release notes generation
  - Commit grouping by conventional commit type
  - Multiple output formats (markdown, JSON, plain)
  - GitHub compare URL generation
  - Merge commit filtering

#### Documentation
- Complete README with examples
- .NET deployment guide with advanced scenarios
- Migration guide for existing projects
- MIT License

### Features

- ✅ Multi-environment support (staging, production)
- ✅ Semantic versioning automation
- ✅ Docker Compose deployment
- ✅ SSH key security validation
- ✅ Health check support
- ✅ Automatic image cleanup
- ✅ Detailed deployment summaries
- ✅ GitHub Release creation
- ✅ Release branch management
- ✅ Conventional commit support

### Technical Details

- GitHub Actions reusable workflows
- Composite actions for modularity
- Bash scripting for portability
- Docker Compose v3.8+ support
- SSH with OpenSSH format keys
- Git tag management
- Semantic versioning (semver 2.0.0)

---

## [Unreleased]

### Planned

- [ ] Support for Kubernetes deployments
- [ ] Slack/Discord notifications
- [ ] Rollback workflows
- [ ] Blue-green deployment support
- [ ] Database migration workflows
- [ ] Terraform integration
- [ ] AWS/Azure deployment workflows
- [ ] Multi-cloud support

---

[1.0.0]: https://github.com/Epilogik/epilogik-shared-workflows/releases/tag/v1.0.0
