# ci-templates

Centralized CI/CD workflows for LuvMatch backend services (`ubiwdotspace`).

## Reusable Workflows

### `golang-deploy-template.yml`

Deploys a Go backend service to Oracle Cloud via SSH. Triggered by child repos using `workflow_call`.

**Flow**: git checkout tag → write `.env` → build → migrate shared PostgreSQL → swap containers → HTTP health check

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `service_name` | ✅ | Systemd service name (e.g., `auth-service`) |
| `deploy_dir` | ✅ | Absolute path on server (e.g., `/var/www/auth-service`) |
| `deploy_tag` | ✅ | Git tag to deploy (e.g., `v1.0.0`) |
| `ssh_host` | ✅ | Target server hostname or IP |
| `compose_file` | | Compose file relative to the deploy directory |
| `postgres_deploy_dir` | ✅ | Deploy directory that owns the shared PostgreSQL service |
| `postgres_compose_file` | | PostgreSQL compose file relative to its deploy directory |
| `database_user` | ✅ | Database role used to apply migrations |
| `database_name` | ✅ | Database receiving the service migrations |
| `healthcheck_url` | ✅ | Health URL reachable from the target server |

**Secrets** (via `secrets: inherit` from Organization)

| Name | Description |
|------|-------------|
| `ORACLE_SSH_HOST` | Public IP of Oracle server |
| `ORACLE_SSH_USER` | SSH username (`ubuntu`) |
| `ORACLE_SSH_KEY` | PEM private key content |
| `SERVICE_ENV` | Repo-specific production `.env` content |
| `SERVICE_CONFIG` | Legacy fallback for `SERVICE_ENV` during migration |

## Usage

In each child repo, create `.github/workflows/deploy.yml`:

```yaml
name: Deploy

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    if: github.ref_type == 'tag' && startsWith(github.ref, 'refs/tags/v')
    uses: ubiwdotspace/ci-templates/.github/workflows/golang-deploy-template.yml@v1.0.0
    with:
      service_name: auth-service
      deploy_dir: /var/www/auth-service
      deploy_tag: ${{ github.ref_name }}
      ssh_host: ${{ vars.DEPLOY_HOST }}
      compose_file: docker-compose.prod.yml
      postgres_deploy_dir: /var/www/auth-service
      postgres_compose_file: docker-compose.prod.yml
      database_user: authorization
      database_name: authorization
      healthcheck_url: http://127.0.0.1:8080/healthz
    secrets: inherit
```

## Adding a New Service

1. Set up server: create deploy dir, systemd unit, sudoers entry
2. Add a repo-level `SERVICE_ENV` secret containing that service's production `.env`
3. Create `.github/workflows/deploy.yml` in the child repo with the correct `service_name` and `deploy_dir`
