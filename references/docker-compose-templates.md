# Docker Compose Templates

## Case A — Single Service, `docker/.env` Only (SPA / static)

```yaml
services:
  <SERVICE_NAME>:
    image: <DOCKERHUB_USER>/<IMAGE>:latest
    container_name: <SERVICE_NAME>
    restart: unless-stopped
    env_file:
      - .env          # docker/.env (relative to this compose file)
    ports:
      - '${CONTAINER_PORT:-80}:80'
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:80']
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Case B — Single Service, App `.env` + `docker/.env`

The `docker-compose.yml` lives in `docker/`, so `../.env` points to the project root `.env`:

```yaml
services:
  <SERVICE_NAME>:
    image: <DOCKERHUB_USER>/<IMAGE>:latest
    container_name: <SERVICE_NAME>
    restart: unless-stopped
    env_file:
      - .env       # docker/.env  — docker-level vars (ports, DB init creds)
      - ../.env    # project .env — app secrets (JWT, API keys, DB URIs)
    ports:
      - '${CONTAINER_PORT:-3000}:3000'
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:3000']
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Case C — Multi-Service with Volumes & Networks (auth pattern)

Full stack: app + MongoDB + Redis + RedisInsight on a shared internal network.

```yaml
volumes:
  <PROJECT>-db_data:
  <PROJECT>-redis_data:
  <PROJECT>-redisinsight_data:

networks:
  <PROJECT>_net:

services:

  # ── MongoDB ──────────────────────────────────────────────────────────────
  <PROJECT>-db:
    image: mongo:latest
    container_name: <PROJECT>-db
    restart: unless-stopped
    volumes:
      - <PROJECT>-db_data:/data/db
    networks:
      - <PROJECT>_net
    env_file:
      - .env          # contains MONGO_INITDB_ROOT_USERNAME / PASSWORD
    healthcheck:
      test:
        [
          'CMD', 'mongosh', '--quiet', '--eval',
          '-u ${MONGO_INITDB_ROOT_USERNAME}',
          '-p ${MONGO_INITDB_ROOT_PASSWORD}',
          'db.runCommand({ ping: 1 }).ok'
        ]
      interval: 10s
      timeout: 5s
      retries: 3

  # ── Redis ─────────────────────────────────────────────────────────────────
  <PROJECT>-redis:
    image: redis:latest
    container_name: <PROJECT>-redis
    restart: unless-stopped
    networks:
      - <PROJECT>_net
    volumes:
      - <PROJECT>-redis_data:/data
    env_file:
      - .env
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 3

  # ── RedisInsight (optional admin UI) ─────────────────────────────────────
  <PROJECT>-redisinsight:
    image: <DOCKERHUB_USER>/redisinsight:latest   # custom image with curl
    container_name: <PROJECT>-redisinsight
    restart: unless-stopped
    networks:
      - <PROJECT>_net
    volumes:
      - <PROJECT>-redisinsight_data:/data
    env_file:
      - .env
    ports:
      - '${CONTAINER_REDISINSIGHT_PORT:-8085}:5540'
    depends_on:
      - <PROJECT>-redis
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:5540/api/health']
      interval: 30s
      timeout: 10s
      retries: 3

  # ── App ───────────────────────────────────────────────────────────────────
  <PROJECT>:
    image: <DOCKERHUB_USER>/<IMAGE>:latest
    container_name: <PROJECT>
    restart: unless-stopped
    depends_on:
      - <PROJECT>-db
    networks:
      - <PROJECT>_net
    env_file:
      - .env       # docker/.env — ports, DB init creds
      - ../.env    # project .env — app secrets
    ports:
      - '${CONTAINER_PORT:-3000}:3000'
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:3000/health/ready']
      interval: 30s
      timeout: 10s
      retries: 3
```

---

## Health Check Patterns Reference

```yaml
# HTTP endpoint (nginx, Next.js, NestJS)
healthcheck:
  test: ['CMD', 'curl', '-f', 'http://localhost:<PORT>']

# Custom API health route
healthcheck:
  test: ['CMD', 'curl', '-f', 'http://localhost:3000/health/ready']

# Redis
healthcheck:
  test: ['CMD', 'redis-cli', 'ping']

# MongoDB
healthcheck:
  test: ['CMD', 'mongosh', '--quiet', '--eval', 'db.runCommand({ ping: 1 }).ok']
```

---

## Naming Conventions

| Resource | Pattern | Example |
|---------|---------|---------|
| Volume | `<project>-<service>_data` | `auth-db_data` |
| Network | `<project>_net` | `auth_net` |
| Container | `<project>-<service>` (or just `<project>` for main) | `auth-db`, `auth` |
| Compose project (`-p`) | project root name | `auth` |

---

## Notes

- `env_file` paths are relative to the **compose file location**, not CWD
- `depends_on` only waits for container start, not health — use `condition: service_healthy` for strict ordering:
  ```yaml
  depends_on:
    <PROJECT>-db:
      condition: service_healthy
  ```
- Named volumes persist across `docker compose down`; use `docker compose down -v` to remove them
