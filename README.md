# ci-templates

Centralized CI/CD workflows for LuvMatch backend services (`ubiwdotspace`).

## Reusable Workflows

### `golang-deploy-template.yml`

Deploys a Go backend service to Oracle Cloud via SSH. Triggered by child repos using `workflow_call`.

**Flow**: git checkout tag → write config → systemctl restart → health check

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `service_name` | ✅ | Systemd service name (e.g., `auth-service`) |
| `deploy_dir` | ✅ | Absolute path on server (e.g., `/var/www/auth-service`) |
| `deploy_tag` | ✅ | Git tag to deploy (e.g., `v1.0.0`) |

**Secrets** (via `secrets: inherit` from Organization)

| Name | Description |
|------|-------------|
| `ORACLE_SSH_HOST` | Public IP of Oracle server |
| `ORACLE_SSH_USER` | SSH username (`ubuntu`) |
| `ORACLE_SSH_KEY` | PEM private key content |
| `SERVICE_CONFIG` | Full `config.yml` content for production (optional, written to server before restart) |

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
    uses: ubiwdotspace/ci-templates/.github/workflows/golang-deploy-template.yml@main
    with:
      service_name: auth-service
      deploy_dir: /var/www/auth-service
      deploy_tag: ${{ github.ref_name }}
    secrets: inherit
```

## Adding a New Service

1. Set up server: create deploy dir, systemd unit, sudoers entry
2. Add `SERVICE_CONFIG` secret at org level (or per-repo if configs differ)
3. Create `.github/workflows/deploy.yml` in the child repo with the correct `service_name` and `deploy_dir`
