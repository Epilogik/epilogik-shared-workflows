# .NET Deployment Examples

This document provides complete examples for deploying .NET applications using Epilogik Shared Workflows.

---

## Basic .NET Web API Deployment

### Project Structure

```
myapp/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── staging/
│   └── docker-compose.yml
├── production/
│   └── docker-compose.yml
├── MyApp.Api/
│   ├── Dockerfile
│   └── MyApp.Api.csproj
└── README.md
```

### GitHub Workflow (`.github/workflows/deploy.yml`)

```yaml
name: Deploy .NET API

on:
  push:
    branches:
      - develop      # Deploy to staging
      - release/**   # Deploy to production
      - main         # Tag and cleanup
  workflow_dispatch:

jobs:
  # ========================================
  # STAGING DEPLOYMENT (develop branch)
  # ========================================
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: staging
      deploy_path: /home/deploy/myapp
      compose_files: docker-compose.yml
      docker_network: myapp-network
    secrets:
      ssh_host: ${{ secrets.STAGING_SSH_HOST }}
      ssh_user: ${{ secrets.STAGING_SSH_USER }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}

  # ========================================
  # CREATE RELEASE PR (after staging)
  # ========================================
  create-release:
    if: github.ref == 'refs/heads/develop'
    needs: deploy-staging
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/create-release-pr.yml@v1
    with:
      base_branch: develop
      bump_type: patch  # or: major, minor, 1.2.3

  # ========================================
  # PRODUCTION DEPLOYMENT (release/* branches)
  # ========================================
  deploy-production:
    if: startsWith(github.ref, 'refs/heads/release/')
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: production
      deploy_path: /home/deploy/myapp
      compose_files: docker-compose.yml
      docker_network: myapp-network
    secrets:
      ssh_host: ${{ secrets.PROD_SSH_HOST }}
      ssh_user: ${{ secrets.PROD_SSH_USER }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}

  # ========================================
  # TAG AND CLEANUP (main branch)
  # ========================================
  tag-release:
    if: github.ref == 'refs/heads/main'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/tag-and-cleanup.yml@v1
    with:
      tag_prefix: v
      create_github_release: true
      delete_release_branch: true
```

### Docker Compose (`staging/docker-compose.yml`)

```yaml
version: '3.8'

services:
  api:
    image: ghcr.io/myorg/myapp-api:develop
    container_name: myapp-api-staging
    restart: unless-stopped
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Staging
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__Default=Server=postgres;Database=myapp_staging;User Id=postgres;Password=${DB_PASSWORD}
    networks:
      - myapp-network
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:16-alpine
    container_name: myapp-postgres-staging
    restart: unless-stopped
    environment:
      - POSTGRES_DB=myapp_staging
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - myapp-network

  redis:
    image: redis:7-alpine
    container_name: myapp-redis-staging
    restart: unless-stopped
    networks:
      - myapp-network

networks:
  myapp-network:
    driver: bridge

volumes:
  postgres_data:
```

---

## Advanced: Multi-Service .NET Application

### Project Structure

```
enterprise-app/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── staging/
│   ├── docker-compose.infra.yml
│   └── docker-compose.services.yml
├── production/
│   ├── docker-compose.infra.yml
│   └── docker-compose.services.yml
├── src/
│   ├── Api/
│   │   ├── Dockerfile
│   │   └── Api.csproj
│   ├── Worker/
│   │   ├── Dockerfile
│   │   └── Worker.csproj
│   └── Blazor/
│       ├── Dockerfile
│       └── Blazor.csproj
└── README.md
```

### Infrastructure Compose (`staging/docker-compose.infra.yml`)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=myapp
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - myapp-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - myapp-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  rabbitmq:
    image: rabbitmq:3-management-alpine
    restart: unless-stopped
    environment:
      - RABBITMQ_DEFAULT_USER=myapp
      - RABBITMQ_DEFAULT_PASS=${RABBITMQ_PASSWORD}
    ports:
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - myapp-network
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 30s
      timeout: 10s
      retries: 5

networks:
  myapp-network:
    external: true

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
```

### Services Compose (`staging/docker-compose.services.yml`)

```yaml
version: '3.8'

services:
  api:
    image: ghcr.io/myorg/myapp-api:${VERSION:-develop}
    restart: unless-stopped
    ports:
      - "5000:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Staging
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__Default=Server=postgres;Database=myapp;User Id=myapp;Password=${DB_PASSWORD}
      - Redis__Configuration=redis:6379,password=${REDIS_PASSWORD}
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=myapp
      - RabbitMQ__Password=${RABBITMQ_PASSWORD}
    networks:
      - myapp-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  worker:
    image: ghcr.io/myorg/myapp-worker:${VERSION:-develop}
    restart: unless-stopped
    environment:
      - DOTNET_ENVIRONMENT=Staging
      - ConnectionStrings__Default=Server=postgres;Database=myapp;User Id=myapp;Password=${DB_PASSWORD}
      - RabbitMQ__Host=rabbitmq
      - RabbitMQ__Username=myapp
      - RabbitMQ__Password=${RABBITMQ_PASSWORD}
    networks:
      - myapp-network
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  blazor:
    image: ghcr.io/myorg/myapp-blazor:${VERSION:-develop}
    restart: unless-stopped
    ports:
      - "5001:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Staging
      - ASPNETCORE_URLS=http://+:8080
      - ApiBaseUrl=http://api:8080
    networks:
      - myapp-network
    depends_on:
      - api

networks:
  myapp-network:
    external: true
```

### Advanced Workflow

```yaml
name: Deploy Enterprise App

on:
  push:
    branches: [develop, release/**, main]
  workflow_dispatch:
    inputs:
      version_bump:
        description: 'Version bump type'
        required: false
        default: 'patch'
        type: choice
        options:
          - major
          - minor
          - patch

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: staging
      deploy_path: /home/deploy/enterprise-app
      compose_files: docker-compose.infra.yml,docker-compose.services.yml
      docker_network: myapp-network
      wait_for_health: true
      cleanup_images: true
    secrets:
      ssh_host: ${{ secrets.STAGING_SSH_HOST }}
      ssh_user: ${{ secrets.STAGING_SSH_USER }}
      ssh_port: ${{ secrets.STAGING_SSH_PORT }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}

  create-release:
    if: github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    needs: [deploy-staging]
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/create-release-pr.yml@v1
    with:
      base_branch: develop
      bump_type: ${{ github.event.inputs.version_bump || 'patch' }}
      auto_merge: false

  deploy-production:
    if: startsWith(github.ref, 'refs/heads/release/')
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/deploy-hostinger.yml@v1
    with:
      environment: production
      deploy_path: /home/deploy/enterprise-app
      compose_files: docker-compose.infra.yml,docker-compose.services.yml
      docker_network: myapp-network
      wait_for_health: true
      cleanup_images: true
    secrets:
      ssh_host: ${{ secrets.PROD_SSH_HOST }}
      ssh_user: ${{ secrets.PROD_SSH_USER }}
      ssh_port: ${{ secrets.PROD_SSH_PORT }}
      ssh_key: ${{ secrets.SSH_PRIVATE_KEY }}

  tag-release:
    if: github.ref == 'refs/heads/main'
    uses: Epilogik/epilogik-shared-workflows/.github/workflows/tag-and-cleanup.yml@v1
    with:
      tag_prefix: v
      create_github_release: true
      delete_release_branch: true
```

---

## Secrets Configuration

### Required Secrets

```bash
# Staging
STAGING_SSH_HOST=185.139.1.15
STAGING_SSH_USER=root
STAGING_SSH_PORT=22

# Production
PROD_SSH_HOST=31.97.173.8
PROD_SSH_USER=root
PROD_SSH_PORT=22

# Shared
SSH_PRIVATE_KEY=<your-private-key>
```

### Optional: Environment Variables

Add to GitHub Environment variables (Settings → Environments → staging/production):

```bash
DB_PASSWORD=<secure-password>
REDIS_PASSWORD=<secure-password>
RABBITMQ_PASSWORD=<secure-password>
VERSION=develop  # or release version
```

---

## Tips & Best Practices

### 1. **Dockerfile Optimization**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.Api/MyApp.Api.csproj", "MyApp.Api/"]
RUN dotnet restore "MyApp.Api/MyApp.Api.csproj"
COPY . .
WORKDIR "/src/MyApp.Api"
RUN dotnet build "MyApp.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApp.Api.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### 2. **Health Checks**

Add to your `Program.cs`:

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(builder.Configuration.GetConnectionString("Default"))
    .AddRedis(builder.Configuration["Redis:Configuration"]);

app.MapHealthChecks("/health");
```

### 3. **Environment-specific Configuration**

Use `appsettings.{Environment}.json`:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

---

## Troubleshooting

### Issue: Container not starting

**Check logs:**
```bash
ssh user@host "docker logs myapp-api-staging"
```

### Issue: Database connection failed

**Verify network:**
```bash
ssh user@host "docker network inspect myapp-network"
```

### Issue: Health check failing

**Check service health:**
```bash
ssh user@host "docker compose ps"
```

---

**Need help?** Open an issue in [epilogik-shared-workflows](https://github.com/Epilogik/epilogik-shared-workflows/issues)
