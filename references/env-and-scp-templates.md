# Env Templates, SCP, Nginx & Deploy Script

## `.env.template` Patterns

### docker/.env.template — Docker-level vars only

```bash
# Port exposed by the container to the host
CONTAINER_PORT=${CONTAINER_PORT}
```

### docker/.env.template — With DB init credentials (multi-service)

```bash
# MongoDB init credentials (used by mongo container)
MONGO_INITDB_ROOT_USERNAME=${MONGO_INITDB_ROOT_USERNAME}
MONGO_INITDB_ROOT_PASSWORD=${MONGO_INITDB_ROOT_PASSWORD}

# Ports
CONTAINER_PORT=${CONTAINER_PORT}
CONTAINER_REDISINSIGHT_PORT=${CONTAINER_REDISINSIGHT_PORT}
```

### .env.template — App-level secrets (NestJS example)

```bash
# App
APP_NAME=${APP_NAME}
TIMEZONE=${TIMEZONE}

# Logger
LOG_LEVEL=${LOG_LEVEL}

# Database
MONGODB_URI=${MONGODB_URI}

# JWT
JWT_ACCESS_SECRET=${JWT_ACCESS_SECRET}
JWT_ACCESS_EXPIRES_IN=${JWT_ACCESS_EXPIRES_IN}
JWT_REFRESH_SECRET=${JWT_REFRESH_SECRET}
JWT_REFRESH_EXPIRES_IN=${JWT_REFRESH_EXPIRES_IN}

# OAuth - Google
GG_CLIENT_ID=${GG_CLIENT_ID}
GG_CLIENT_SECRET=${GG_CLIENT_SECRET}
GG_REDIRECT_URI=${GG_REDIRECT_URI}

# OAuth - GitHub
GH_CLIENT_ID=${GH_CLIENT_ID}
GH_CLIENT_SECRET=${GH_CLIENT_SECRET}
GH_REDIRECT_URI=${GH_REDIRECT_URI}

# Redis
REDIS_URI=${REDIS_URI}
```

### .env.template — Frontend build vars (Vite)

```bash
VITE_API_BASE_URL=${VITE_API_BASE_URL}
```

> Vite build-time vars are injected via `build-args:` in the workflow, NOT via .env on VPS. The `.env.template` / SCP pattern is only needed for runtime vars.

---

## envsubst Workflow Steps

### Single .env.template

```yaml
- name: Install envsubst
  run: |
    sudo apt update
    sudo apt install -y gettext-base

- name: Prepare docker environment file on runner
  env:
    CONTAINER_PORT: ${{ secrets.CONTAINER_PORT }}
  run: |
    envsubst < docker/.env.template > docker/.env
```

### Dual .env.template (app + docker)

```yaml
- name: Install envsubst
  run: |
    sudo apt update
    sudo apt install -y gettext-base

- name: Prepare application environment file on runner
  env:
    APP_NAME: ${{ secrets.APP_NAME }}
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
    JWT_ACCESS_SECRET: ${{ secrets.JWT_ACCESS_SECRET }}
    # … all app vars
  run: |
    envsubst < .env.template > .env

- name: Prepare docker environment file on runner
  env:
    CONTAINER_PORT: ${{ secrets.CONTAINER_PORT }}
    MONGO_INITDB_ROOT_USERNAME: ${{ secrets.MONGO_INITDB_ROOT_USERNAME }}
    MONGO_INITDB_ROOT_PASSWORD: ${{ secrets.MONGO_INITDB_ROOT_PASSWORD }}
  run: |
    envsubst < docker/.env.template > docker/.env
```

---

## SCP Action Full Options

```yaml
- name: Copy environment file
  uses: appleboy/scp-action@v1
  with:
    host: ${{ secrets.VPS_HOST }}
    port: ${{ secrets.VPS_PORT }}
    username: ${{ secrets.VPS_USERNAME }}
    key: ${{ secrets.VPS_SSH_KEY }}

    # Source (required)
    source: 'docker/.env'
    # Multiple files: 'docker/.env,.env,scripts/'

    # Destination directory on VPS (required)
    target: '/home/user/app'

    # Optional:
    strip_components:
      1 # strip N leading path components
      # 'docker/.env' with strip=1 → copied as '.env', not 'docker/.env'
    rm: false # delete all files in target dir before copy
    overwrite: true # overwrite existing files (default false)
    timeout: '30s' # SSH connection timeout
    command_timeout: '10m' # transfer command timeout
```

### strip_components use cases

```
source='docker/.env',      strip=0 → target/docker/.env  (default)
source='docker/.env',      strip=1 → target/.env
source='config/app/db.yml', strip=2 → target/db.yml
```

Use `strip_components: 1` when target dir IS the docker/ directory on VPS, to avoid nesting.

---

## nginx.conf — SPA (React Router / Vue Router)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA fallback: unknown paths serve index.html (client-side router handles them)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Aggressively cache static assets (hashed filenames)
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/javascript
        application/javascript
        application/json
        image/svg+xml;
}
```

---

## scripts/deploy.sh — Full Template

```bash
#!/usr/bin/env bash
set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

if [ $# -eq 0 ]; then
  echo -e "${RED}Error: Environment parameter is required${NC}"
  echo -e "${YELLOW}Usage: yarn deploy:develop  OR  yarn deploy:production${NC}"
  exit 1
fi

ENV=$1
if [ "$ENV" != "develop" ] && [ "$ENV" != "production" ]; then
  echo -e "${RED}Error: Environment must be 'develop' or 'production'${NC}"
  exit 1
fi

# ── Local deploy (build + run docker locally) ─────────────────────────────
# Uncomment and adapt if you want a local mode:
if [ "$ENV" == "local" ]; then
  cd "$(dirname "$0")/.." || exit 1
  if [ ! -f .env ] || [ ! -f docker/.env ]; then
    echo -e "${RED}Missing .env or docker/.env${NC}"; exit 1
  fi
  docker build -t <DOCKERHUB_USER>/<IMAGE>:latest -f ./docker/Dockerfile .
  docker compose -f docker/docker-compose.yml -p <PROJECT> down
  docker compose -f docker/docker-compose.yml -p <PROJECT> up -d --remove-orphans
  exit 0
fi

TARGET_BRANCH=$ENV

if ! git diff --quiet || ! git diff --cached --quiet; then
  echo -e "${RED}Error: Uncommitted changes. Commit or stash first.${NC}"
  exit 1
fi

# Loading indicator while git operations run in background
show_loading() {
  local pid=$1
  local count=0
  local max_dots=30
  local dots=""
  while kill -0 $pid 2>/dev/null; do
    count=$((count + 1))
    if [ $count -gt $max_dots ]; then count=1; dots=""; printf "\r${YELLOW}%${max_dots}s${NC}" ""; fi
    dots="${dots}."
    printf "\r${YELLOW}${dots}${NC}"
    sleep 0.75
  done
  printf "\r%${max_dots}s\r" ""
}

echo -e "${YELLOW}Starting deployment for: ${ENV}${NC}"

CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
{
  git fetch --prune origin ${TARGET_BRANCH} >/dev/null 2>&1
  git branch -D ${TARGET_BRANCH} >/dev/null 2>&1 || true
  git checkout -b ${TARGET_BRANCH} origin/${TARGET_BRANCH} >/dev/null 2>&1 || true

  # Empty commit — message pattern triggers the GitHub Actions workflow
  commit_message="ci: deploy to ${ENV} - $(date +'%H:%M:%S %d/%m/%Y')"
  git commit --allow-empty -m "${commit_message}" >/dev/null 2>&1
} &
DEPLOY_PID=$!
show_loading $DEPLOY_PID
wait $DEPLOY_PID || true

git push origin ${TARGET_BRANCH}
git checkout ${CURRENT_BRANCH} >/dev/null 2>&1

echo -e "${GREEN}========================================${NC}"
echo -e "${GREEN}Deployment trigger completed!${NC}"
echo -e "${GREEN}GitHub Actions will handle the rest.${NC}"
echo -e "${GREEN}Environment: ${ENV} | Branch: ${TARGET_BRANCH}${NC}"
echo -e "${GREEN}========================================${NC}"
```

## package.json deploy scripts

```json
{
	"scripts": {
		"deploy:local": "sh scripts/deploy.sh local",
		"deploy:develop": "sh scripts/deploy.sh develop",
		"deploy:production": "sh scripts/deploy.sh production"
	}
}
```

> For monorepo root or bun-based projects, use `bun deploy:develop` / `bun deploy:production`.
