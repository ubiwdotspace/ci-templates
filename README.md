# ci-templates

Centralized CI/CD workflows for LuvMatch backend services (`ubiwdotspace`).

## Reusable Workflows

### `docker-build-ghcr-template.yml`

Builds Docker container images for PR checks and GHCR publishing through the
same implementation. `push: false` builds without logging in or publishing;
the default `push: true` publishes to `ghcr.io/ubiwdotspace/...`. Published
images keep the release tag, commit SHA (`sha-...`), and tag-triggered `latest`.

GitHub Actions build cache is scoped by image name and API so parallel REST
and gRPC builds do not overwrite each other's cache. Existing callers need
no new inputs; the first build after upgrading starts a new cache scope.

**Inputs**:
* `image_name` (required): Target image name in GHCR (e.g., `auth-service-rest`).
* `build_arg_api`: Target binary to compile (`rest` or `grpc`). Defaults to `rest`.
* `platforms`: Target platforms for buildx (defaults to `linux/arm64` for Oracle Ampere A1).
* `dockerfile`: Path to Dockerfile (defaults to `Dockerfile`).
* `push`: Publish to GHCR (defaults to `true`; set `false` for PR builds).
* `runner`: Compatible runner label (defaults to `ubuntu-latest`). Ubuntu is
  convenient, but a Linux self-hosted runner with Docker can also be used.
* `timeout_minutes`: Job timeout (defaults to GitHub's existing 360-minute
  limit; callers can explicitly use a shorter limit such as `15`).

Callers control token permissions: use `contents: read` for build-only jobs,
and add `packages: write` for publishing. The template inherits those
permissions and uses the caller's `GITHUB_TOKEN`; do not inherit runtime
secrets for image builds.

Both modes use `docker/build-push-action@v6`, which produces GitHub job logs,
timings, a build summary, and a downloadable `.dockerbuild` record by default.
Record retention follows the repository/organization policy. This record
contains build diagnostics, not an image archive; published images live in
GHCR, and PR builds do not upload an image tarball.

Service repositories keep their event triggers and image/API configuration;
checkout, Buildx/QEMU, cache, build, optional login/push, and build records
belong here. Pin callers to a published version tag. The current review
candidate is `v1.3.0-rc.1`; do not move existing version tags.

### `golang-deploy-template.yml`

Deploys a Go backend service to Oracle Cloud via SSH. Triggered by child repos using `workflow_call`. Database migrations can be disabled for stateless services, and health checks support HTTP or the standard gRPC health service.

**Flow**: git checkout tag → write `.env` → build → migrate shared PostgreSQL → swap containers → HTTP health check

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `service_name` | ✅ | Systemd service name (e.g., `auth-service`) |
| `deploy_dir` | ✅ | Absolute path on server (e.g., `/var/www/auth-service`) |
| `deploy_tag` | ✅ | Git tag to deploy (e.g., `v1.0.0`) |
| `ssh_host` | ✅ | Target server hostname or IP |
| `compose_file` | | Compose file relative to the deploy directory |
| `postgres_deploy_dir` | When migrations are enabled | Deploy directory that owns the shared PostgreSQL service |
| `postgres_compose_file` | | PostgreSQL compose file relative to its deploy directory |
| `database_user` | When migrations are enabled | Database role used to apply migrations |
| `database_name` | When migrations are enabled | Database receiving the service migrations |
| `run_migrations` | | Apply SQL migrations (defaults to `true`) |
| `healthcheck_url` | For HTTP checks | Health URL reachable from the target server |
| `healthcheck_type` | | `http` (default) or `grpc`; gRPC checks run `./healthcheck` in the `grpc` compose service |

**Secrets** (via `secrets: inherit` from Organization)

| Name | Description |
|------|-------------|
| `ORACLE_SSH_HOST` | Public IP of Oracle server |
| `ORACLE_SSH_USER` | SSH username (`ubuntu`) |
| `ORACLE_SSH_KEY` | PEM private key content |
| `SERVICE_ENV` | Optional repo-specific production `.env` content; when absent, the existing server `.env` is kept |
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

Stateless gRPC services can set `run_migrations: false` and `healthcheck_type: grpc`. Their `grpc` container must include an executable `./healthcheck` that checks the standard gRPC health service.

## Adding a New Service

1. Set up server: create deploy dir, systemd unit, sudoers entry
2. Add a repo-level `SERVICE_ENV` secret containing that service's production `.env`, or provision the `.env` on the server over SSH
3. Create `.github/workflows/deploy.yml` in the child repo with the correct `service_name` and `deploy_dir`
