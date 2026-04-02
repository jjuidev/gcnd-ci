---
name: gcnd-ci
description: CI/CD flow for GitHub Actions → Docker Hub → VPS (Nginx/Docker). Use when setting up deployment pipelines, writing GitHub Actions workflows, Dockerfiles (React/Vite SPA, Next.js, NestJS), docker-compose with volumes/networks, env templating (envsubst), SCP file transfer, or deploy scripts. Handles single-service and multi-service stacks, dual .env files (app + docker), and commit-message-triggered deploys.
---

# GCND-CI: GitHub → Nginx → Docker CI Flow

## Architecture

```
Developer → scripts/deploy.sh → git push to develop/production
         → GitHub Actions triggered (commit message filter)
         → envsubst renders .env templates from GitHub Secrets
         → Docker build + push to Docker Hub
         → SSH into VPS → git pull latest
         → SCP copy generated .env files
         → docker compose pull + down + up
```

## Trigger Pattern

Workflows only fire when commit message contains `deploy to {branch} -`:
```yaml
if: ${{ contains(github.event.head_commit.message, format('deploy to {0} - ', github.ref_name)) }}
```

`scripts/deploy.sh` creates this empty commit automatically. **Never push manually to trigger CI.**

## Environment File Strategy

| File | Purpose | How populated |
|------|---------|--------------|
| `.env.template` | App secrets (JWT, DB URIs, OAuth…) | `envsubst` on runner → `.env` |
| `docker/.env.template` | Docker vars (ports, DB init creds) | `envsubst` on runner → `docker/.env` |
| `.env` | App env on VPS | SCP after generation |
| `docker/.env` | Docker env on VPS | SCP after generation |

When app has both files, docker-compose loads both:
```yaml
env_file:
  - .env        # docker-specific
  - ../.env     # app-level (one dir up from docker/)
```

## Project File Structure

```
project/
├── .env.template              # App env (envsubst placeholders)
├── .github/workflows/
│   └── deploy.yml
├── docker/
│   ├── .env.template          # Docker env (envsubst placeholders)
│   ├── Dockerfile             # Multi-stage with named targets
│   ├── docker-compose.yml
│   └── nginx.conf             # SPA only
└── scripts/
    └── deploy.sh              # Trigger: creates empty commit + push
```

## References

- `references/workflow-templates.md` — GitHub Actions: Case A (docker/.env only), Case B (app+docker .env), Case C (multi-service multi-image)
- `references/dockerfile-templates.md` — Vite SPA, Next.js SSR, NestJS API, React CRA, generic Node
- `references/docker-compose-templates.md` — Single service, multi-service (volumes + networks)
- `references/env-and-scp-templates.md` — .env.template patterns, envsubst steps, SCP options, nginx.conf, deploy.sh

## Setup Checklist

1. Create `.env.template` + `docker/.env.template` with `${VAR_NAME}` syntax
2. Add all vars as GitHub Secrets in `develop`/`production` Environments
3. Write `docker/Dockerfile` — named targets matching workflow `target:` field
4. Write `docker/docker-compose.yml` — `env_file` both `.env` files if needed
5. Write `.github/workflows/deploy.yml` — `environment: ${{ github.ref_name }}`
6. Write `scripts/deploy.sh` — creates empty commit with trigger message
7. Add `deploy:develop` / `deploy:production` to `package.json` scripts

## Common Secrets

```
# VPS SSH
VPS_HOST, VPS_PORT, VPS_USERNAME, VPS_SSH_KEY

# Docker Hub
DOCKER_HUB_USERNAME, DOCKER_HUB_TOKEN

# Port
CONTAINER_PORT
```

## Security Rules

- `chmod 600 .env docker/.env` on VPS after SCP
- Never commit `.env` — only `.env.template` to git
- `docker logout` after pull on VPS
- SSH key auth only (no passwords in workflows)

---

Does NOT handle: Cloudflare Workers, Kubernetes, CI test pipelines, DB migrations.
