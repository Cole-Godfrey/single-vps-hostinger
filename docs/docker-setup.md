# Docker Setup

The production stack is defined by:

- `Dockerfile`
- `docker-compose.prod.yml`
- `Caddyfile`
- `.env`
- `.deploy.env`

## Images

The `Dockerfile` builds two production targets.

### `runner`

Runs the Next.js app.

Build command:

```bash
docker buildx build --platform linux/amd64 --target runner --build-arg DEPLOYMENT_VERSION="$IMAGE_TAG" -t "$APP_IMAGE" --push .
```

### `migrator`

Runs Prisma migrations.

Build command:

```bash
docker buildx build --platform linux/amd64 --target migrator -t "$MIGRATION_IMAGE" --push .
```

## Production Services

`docker-compose.prod.yml` starts:

- `postgres`: private database
- `redis`: private cache
- `app`: Next.js app
- `migrate`: one-off Prisma migration runner
- `uptime-kuma`: status monitoring
- `caddy`: public HTTPS reverse proxy

Only Caddy publishes ports `80` and `443`.

## Environment Files

`.env` contains runtime configuration:

```env
APP_DOMAIN=your-domain.com
STATUS_DOMAIN=status.your-domain.com
BETTER_AUTH_URL=https://your-domain.com
DATABASE_URL=postgresql://user:password@postgres:5432/db?schema=public
REDIS_URL=redis://redis:6379
```

`.deploy.env` contains image tags:

```env
APP_IMAGE=ghcr.io/your-username/hosting-vercel-local-hostinger:app-tag
MIGRATION_IMAGE=ghcr.io/your-username/hosting-vercel-local-hostinger:migrate-tag
```

## Normal Deploy Order

Run on the VPS:

```bash
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml pull
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d postgres redis uptime-kuma
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml run --rm migrate
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d app caddy
```

## Update Only The App

After pushing a new `APP_IMAGE`, update `.deploy.env` on the VPS and run:

```bash
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml pull app
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d --force-recreate app
```

## Inspect The Stack

```bash
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml ps
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml logs --tail=100 app
```
