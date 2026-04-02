# Dockerfile Templates

## 1. Vite / React SPA → Nginx (mns pattern)

Build with bun, serve with nginx. Pass build-time env vars as `ARG`.

```dockerfile
FROM node:20-alpine AS build-env

# Build-time env vars (injected via --build-arg in docker build)
ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL

WORKDIR /app

COPY package.json bun.lock ./
RUN apk add --no-cache curl bash unzip
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:$PATH"
RUN bun install --frozen-lockfile

COPY . .
RUN bun run build

# ---- Runtime: nginx ----
FROM nginx:alpine AS <SERVICE_NAME>

COPY --from=build-env /app/dist /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

> With yarn/npm instead of bun, replace the bun install block with:
> ```dockerfile
> RUN npm install --frozen-lockfile
> # or
> RUN yarn install --frozen-lockfile
> ```

---

## 2. React CRA → Nginx

Same pattern as Vite SPA, but build output is `/app/build`:

```dockerfile
FROM node:20-alpine AS build-env

ARG REACT_APP_API_URL
ENV REACT_APP_API_URL=$REACT_APP_API_URL

WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline
COPY . .
RUN yarn build

FROM nginx:alpine AS <SERVICE_NAME>
COPY --from=build-env /app/build /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

---

## 3. Next.js SSR — 4-Stage (blog pattern)

Separate dev-deps (build) vs prod-deps (runtime) for lean final image.

```dockerfile
FROM node:20-alpine AS development-dependencies-env
WORKDIR /app
COPY ./package.json ./yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline

FROM node:20-alpine AS production-dependencies-env
WORKDIR /app
COPY ./package.json ./yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline --omit=dev

FROM node:20-alpine AS build-env
WORKDIR /app
COPY . .
COPY --from=development-dependencies-env /app/node_modules /app/node_modules
RUN yarn build

FROM node:20-alpine AS <SERVICE_NAME>
WORKDIR /app
RUN apk add --no-cache curl
COPY ./package.json .
COPY --from=production-dependencies-env /app/node_modules /app/node_modules
COPY --from=build-env /app/.next /app/.next
COPY --from=build-env /app/public /app/public
CMD ["yarn", "start"]
EXPOSE 3000
```

---

## 4. NestJS API — 4-Stage (auth pattern)

```dockerfile
FROM node:20-alpine AS development-dependencies-env
WORKDIR /app
COPY ./package.json ./yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline

FROM node:20-alpine AS production-dependencies-env
WORKDIR /app
COPY ./package.json ./yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline --omit=dev

FROM node:20-alpine AS build-env
WORKDIR /app
COPY . .
COPY --from=development-dependencies-env /app/node_modules /app/node_modules
RUN yarn build

FROM node:20-alpine AS <SERVICE_NAME>
WORKDIR /app
RUN apk add --no-cache curl
COPY ./package.json .
COPY --from=production-dependencies-env /app/node_modules /app/node_modules
COPY --from=build-env /app/dist /app/dist
CMD ["yarn", "start:prod"]
EXPOSE 3000
```

---

## 5. Generic Node.js API (Express, Fastify, etc.)

```dockerfile
FROM node:20-alpine AS production-dependencies-env
WORKDIR /app
COPY ./package.json ./yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline --omit=dev

FROM node:20-alpine AS build-env
WORKDIR /app
COPY ./package.json ./yarn.lock ./
RUN yarn install --frozen-lockfile --prefer-offline
COPY . .
RUN yarn build   # or tsc

FROM node:20-alpine AS <SERVICE_NAME>
WORKDIR /app
RUN apk add --no-cache curl
COPY ./package.json .
COPY --from=production-dependencies-env /app/node_modules /app/node_modules
COPY --from=build-env /app/dist /app/dist
CMD ["node", "dist/index.js"]
EXPOSE 3000
```

---

## 6. Multi-Target: Third-Party Image Wrapper (redisinsight pattern)

When you need to push a customised version of a third-party image (e.g. add curl for healthcheck):

```dockerfile
FROM redis/redisinsight:latest AS redisinsight
USER root
RUN apk add --no-cache curl
# No CMD needed — inherits from base image
```

This lets the CI build + push it to your Docker Hub so the VPS pulls a pinned image rather than reaching Docker Hub at runtime.

---

## Notes

- Always name your final stage after the service (`AS <SERVICE_NAME>`) — the workflow's `target:` field references it
- `--omit=dev` requires yarn ≥ 1.x (equiv. `--production` in older yarn)
- For pnpm replace `yarn.lock` with `pnpm-lock.yaml` and `RUN pnpm install --frozen-lockfile`
- For bun replace with `bun.lock` and `RUN bun install --frozen-lockfile`
