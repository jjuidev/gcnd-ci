# GitHub Actions Workflow Templates

## Case A — Single Service, `docker/.env` Only (SPA / Vite pattern)

```yaml
name: Deployment

on:
  push:
    branches:
      - develop
      - production

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.ref_name }}
    if: ${{ contains(github.event.head_commit.message, format('deploy to {0} - ', github.ref_name)) }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install envsubst
        run: |
          sudo apt update
          sudo apt install -y gettext-base

      - name: Prepare docker environment file on runner
        env:
          CONTAINER_PORT: ${{ secrets.CONTAINER_PORT }}
          # Add more docker-level vars here
        run: |
          envsubst < docker/.env.template > docker/.env

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./docker/Dockerfile
          target: <SERVICE_NAME>           # matches Dockerfile stage name
          push: true
          tags: <DOCKERHUB_USER>/<IMAGE>:latest
          build-args: |
            VITE_API_BASE_URL=${{ secrets.VITE_API_BASE_URL }}
          cache-from: type=registry,ref=<DOCKERHUB_USER>/<IMAGE>:latest
          cache-to: type=inline

      - name: Update repository on VPS
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          port: ${{ secrets.VPS_PORT }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            ssh-keyscan -H github.com >> ~/.ssh/known_hosts

            cd /home/<VPS_USER>/<VPS_BASE_DIR> || exit 1

            if [ -d <PROJECT>/.git ]; then
              cd <PROJECT>
              git fetch --prune origin ${{ github.ref_name }} >/dev/null 2>&1
              git reset --hard origin/${{ github.ref_name }} >/dev/null 2>&1
              git branch -D ${{ github.ref_name }} 2>/dev/null || true
              git checkout -b ${{ github.ref_name }} origin/${{ github.ref_name }} 2>/dev/null || true
            else
              git clone -b ${{ github.ref_name }} git@github.com-<HOST_ALIAS>:<ORG>/<REPO>.git <PROJECT> || exit 1
              cd <PROJECT>
              git config pull.rebase false
              git config user.name "<GIT_USER>"
              git config user.email "<GIT_EMAIL>"
            fi

      - name: Copy docker environment file
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          port: ${{ secrets.VPS_PORT }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          source: 'docker/.env'
          target: '/home/<VPS_USER>/<VPS_BASE_DIR>/<PROJECT>'

      - name: Start Docker containers
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.VPS_HOST }}
          port: ${{ secrets.VPS_PORT }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /home/<VPS_USER>/<VPS_BASE_DIR>/<PROJECT> || exit 1

            chmod 600 docker/.env

            echo "${{ secrets.DOCKER_HUB_TOKEN }}" | docker login -u "${{ secrets.DOCKER_HUB_USERNAME }}" --password-stdin || exit 1
            docker compose -f docker/docker-compose.yml -p <PROJECT> pull
            docker compose -f docker/docker-compose.yml -p <PROJECT> down
            docker compose -f docker/docker-compose.yml -p <PROJECT> up -d --remove-orphans
            docker logout
```

---

## Case B — Single Service, App `.env` + `docker/.env` (Next.js / NestJS simple)

Add these steps **after** `Install envsubst`, before Docker login:

```yaml
      - name: Prepare application environment file on runner
        env:
          # App-level secrets — map each GitHub Secret to its env var name
          APP_NAME: ${{ secrets.APP_NAME }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          JWT_SECRET: ${{ secrets.JWT_SECRET }}
          # … add all app vars here
        run: |
          envsubst < .env.template > .env

      - name: Prepare docker environment file on runner
        env:
          CONTAINER_PORT: ${{ secrets.CONTAINER_PORT }}
        run: |
          envsubst < docker/.env.template > docker/.env
```

Add two SCP steps (copy both `.env` files to VPS):

```yaml
      - name: Copy application environment file
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          port: ${{ secrets.VPS_PORT }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          source: '.env'
          target: '/home/<VPS_USER>/<VPS_BASE_DIR>/<PROJECT>'

      - name: Copy docker environment file
        uses: appleboy/scp-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          port: ${{ secrets.VPS_PORT }}
          username: ${{ secrets.VPS_USERNAME }}
          key: ${{ secrets.VPS_SSH_KEY }}
          source: 'docker/.env'
          target: '/home/<VPS_USER>/<VPS_BASE_DIR>/<PROJECT>'
```

VPS start script must `chmod` both:

```bash
chmod 600 .env
chmod 600 docker/.env
```

---

## Case C — Multi-Service, Multi-Image (auth pattern: NestJS + MongoDB + Redis + RedisInsight)

Replace the single build step with multiple `docker/build-push-action` steps:

```yaml
      - name: Build and push Docker image redisinsight
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./docker/Dockerfile
          target: redisinsight
          push: true
          tags: <DOCKERHUB_USER>/redisinsight:latest
          cache-from: type=registry,ref=<DOCKERHUB_USER>/redisinsight:latest
          cache-to: type=inline

      - name: Build and push Docker image <SERVICE>
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./docker/Dockerfile
          target: <SERVICE>
          push: true
          tags: <DOCKERHUB_USER>/<SERVICE>:latest
          cache-from: type=registry,ref=<DOCKERHUB_USER>/<SERVICE>:latest
          cache-to: type=inline
```

Third-party images (mongo, redis) are NOT built — they come from Docker Hub directly in docker-compose.

---

## SCP Action Options

```yaml
- uses: appleboy/scp-action@v1
  with:
    host: ${{ secrets.VPS_HOST }}
    port: ${{ secrets.VPS_PORT }}
    username: ${{ secrets.VPS_USERNAME }}
    key: ${{ secrets.VPS_SSH_KEY }}
    source: 'docker/.env'                  # single file
    # source: 'docker/.env,scripts/'       # multiple files/dirs (comma-separated)
    target: '/home/user/app'               # destination directory on VPS

    # Optional flags:
    strip_components: 1      # strip N leading path components from source
                             # e.g. source='docker/.env', strip=1 → copies as '.env' not 'docker/.env'
    rm: false                # remove all files in target before copy (default: false)
    overwrite: true          # overwrite existing files (default: false)
    timeout: '30s'           # connection timeout
    command_timeout: '10m'   # transfer timeout
```

**`strip_components` example:**
- `source: 'docker/.env'`, `strip_components: 1` → file lands at `target/.env` (not `target/docker/.env`)
- Useful when target dir already IS the docker/ dir on VPS
