# gcnd-ci

Claude Code skill for **GitHub Actions → Docker Hub → VPS (Nginx/Docker)** CI/CD pipelines.

## What It Does

Helps Claude generate production-ready CI/CD configs:

- GitHub Actions `deploy.yml` (commit-message-triggered deploys)
- Multi-stage `Dockerfile` templates (React/Vite SPA, Next.js, NestJS, Node)
- `docker-compose.yml` with volumes, networks, health checks
- Environment file templating with `envsubst`
- `scripts/deploy.sh` deploy trigger scripts

## Patterns Covered

| Case | Services | Env Files |
|------|----------|-----------|
| A — Simple SPA | 1 (nginx) | `docker/.env` only |
| B — Node app | 1 (node) | `.env` + `docker/.env` |
| C — Full stack | 4+ (app + mongo + redis + tools) | `.env` + `docker/.env` |

## Install

```bash
# Via Claude Code skill install
/install-skill gcnd-ci
```
