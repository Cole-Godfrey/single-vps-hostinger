# Single VPS Hostinger

A self-hosted Next.js app for a single VPS. The stack runs Next.js, Postgres, Redis, Caddy, and Uptime Kuma with Docker Compose.

## Stack

- Next.js `16`
- Better Auth
- Prisma and Postgres
- Redis
- Caddy for HTTPS
- Docker Compose for production
- Uptime Kuma for status monitoring

## Local Development

Install dependencies and generate the Prisma client:

```bash
pnpm install
pnpm prisma:generate
```

Create `.env` from the example and set real local values:

```bash
cp .env.example .env
```

Start the app:

```bash
npm run dev
```

Open `http://localhost:3000`.

## Production Deploy

Use the full VPS guide:

```text
docs/self-host-guide.md
```

The short version:

1. Build and push the app image.
2. Build and push the migration image when migrations change.
3. Update `.deploy.env` on the VPS with the image tags.
4. Pull images on the VPS.
5. Run migrations.
6. Recreate the app container.

## Update The Live App

For frontend or app-code changes that do not include database migrations:

```bash
export IMAGE_TAG="$(git rev-parse --short HEAD)-$(date +%Y%m%d%H%M)"
export GHCR_USERNAME="your-username"
export IMAGE_ROOT="ghcr.io/$GHCR_USERNAME/hosting-vercel-local-hostinger"
export APP_IMAGE="${IMAGE_ROOT}:app-${IMAGE_TAG}"

docker buildx build --platform linux/amd64 --target runner --build-arg DEPLOYMENT_VERSION="$IMAGE_TAG" -t "$APP_IMAGE" --push .
echo "$APP_IMAGE"
```

On the VPS, put the echoed value in `.deploy.env` as `APP_IMAGE`, then run:

```bash
cd /srv/hosting-vercel-local-hostinger
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml pull app
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d --force-recreate app
```

## Useful Commands

```bash
npm run lint
npm run build
pnpm exec prisma validate
```

## Credit

Thank you to [ski043](https://github.com/ski043) for his contributions to this project.
