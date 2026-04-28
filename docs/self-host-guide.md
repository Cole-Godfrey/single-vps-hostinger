# Self-Host Guide

This guide deploys the app to one Ubuntu VPS with Docker Compose.

## What You Need

- Ubuntu `24.04` VPS
- Domain and status subdomain
- Docker running locally
- GitHub Container Registry push token
- GitHub Container Registry read token for the VPS

## Local Values

Run locally from the repo root:

```bash
export VPS_IP="your-vps-ip"
export VPS_USER="deploy"
export APP_DIR="/srv/hosting-vercel-local-hostinger"
export APP_DOMAIN="your-domain.com"
export STATUS_DOMAIN="status.your-domain.com"
export GHCR_USERNAME="your-username"
export GHCR_TOKEN="token-with-package-write"
export GHCR_READ_TOKEN="token-with-package-read"
export IMAGE_TAG="$(git rev-parse --short HEAD)"
export IMAGE_ROOT="ghcr.io/$GHCR_USERNAME/hosting-vercel-local-hostinger"
export APP_IMAGE="${IMAGE_ROOT}:app-${IMAGE_TAG}"
export MIGRATION_IMAGE="${IMAGE_ROOT}:migrate-${IMAGE_TAG}"
```

## 1. Prepare DNS

Create these `A` records:

```text
your-domain.com        -> your VPS IP
status.your-domain.com -> your VPS IP
```

## 2. Create The Deploy User

SSH into the VPS as `root`:

```bash
ssh root@$VPS_IP
```

Run:

```bash
adduser deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

Reconnect as `deploy`:

```bash
ssh deploy@$VPS_IP
```

## 3. Install Docker

Run on the VPS:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker deploy
```

Log out and back in, then verify:

```bash
docker version
docker compose version
```

## 4. Open Firewall Ports

Run on the VPS:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status
```

## 5. Create The App Directory

Run on the VPS:

```bash
sudo mkdir -p /srv/hosting-vercel-local-hostinger
sudo chown -R deploy:deploy /srv/hosting-vercel-local-hostinger
cd /srv/hosting-vercel-local-hostinger
```

Copy the production files from your local machine:

```bash
scp docker-compose.prod.yml Caddyfile $VPS_USER@$VPS_IP:$APP_DIR/
scp .env.example $VPS_USER@$VPS_IP:$APP_DIR/
```

## 6. Create The VPS `.env`

Run on the VPS:

```bash
cd /srv/hosting-vercel-local-hostinger
cp .env.example .env
nano .env
```

Set these values:

```env
APP_DOMAIN=your-domain.com
STATUS_DOMAIN=status.your-domain.com
LETSENCRYPT_EMAIL=ops@example.com
BETTER_AUTH_URL=https://your-domain.com
BETTER_AUTH_SECRET=replace-with-a-long-random-secret
NEXT_SERVER_ACTIONS_ENCRYPTION_KEY=replace-with-a-base64-key
POSTGRES_DB=hostgodfrey
POSTGRES_USER=hostgodfrey
POSTGRES_PASSWORD=replace-with-a-strong-password
DATABASE_URL=postgresql://hostgodfrey:replace-with-a-strong-password@postgres:5432/hostgodfrey?schema=public
REDIS_URL=redis://redis:6379
```

Generate secrets with:

```bash
openssl rand -hex 32
openssl rand -base64 32
```

## 7. Build And Push Images

Run locally:

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GHCR_USERNAME" --password-stdin
docker buildx build --platform linux/amd64 --target runner --build-arg DEPLOYMENT_VERSION="$IMAGE_TAG" -t "$APP_IMAGE" --push .
docker buildx build --platform linux/amd64 --target migrator -t "$MIGRATION_IMAGE" --push .
```

## 8. Configure Image Tags On The VPS

Run on the VPS:

```bash
cd /srv/hosting-vercel-local-hostinger
nano .deploy.env
```

Paste the full image names you pushed:

```env
APP_IMAGE=ghcr.io/your-username/hosting-vercel-local-hostinger:app-tag
MIGRATION_IMAGE=ghcr.io/your-username/hosting-vercel-local-hostinger:migrate-tag
```

## 9. Start The Stack

Run on the VPS:

```bash
export GHCR_USERNAME="your-username"
export GHCR_READ_TOKEN="token-with-package-read"
echo "$GHCR_READ_TOKEN" | docker login ghcr.io -u "$GHCR_USERNAME" --password-stdin
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml pull
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d postgres redis uptime-kuma
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml run --rm migrate
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d app caddy
```

Open:

```text
https://your-domain.com
https://status.your-domain.com
```

## Update The App Later

For app changes without migrations, build a new app image locally:

```bash
export IMAGE_TAG="$(git rev-parse --short HEAD)-$(date +%Y%m%d%H%M)"
export GHCR_USERNAME="your-username"
export IMAGE_ROOT="ghcr.io/$GHCR_USERNAME/hosting-vercel-local-hostinger"
export APP_IMAGE="${IMAGE_ROOT}:app-${IMAGE_TAG}"
docker buildx build --platform linux/amd64 --target runner --build-arg DEPLOYMENT_VERSION="$IMAGE_TAG" -t "$APP_IMAGE" --push .
echo "$APP_IMAGE"
```

On the VPS, set `APP_IMAGE` in `.deploy.env` to the echoed value, then run:

```bash
cd /srv/hosting-vercel-local-hostinger
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml pull app
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml up -d --force-recreate app
```

For schema changes, also build and push the `migrator` target, update `MIGRATION_IMAGE`, and run the migration command before recreating the app.

## Useful Checks

```bash
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml ps
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml logs --tail=100 app
docker compose --env-file .env --env-file .deploy.env -f docker-compose.prod.yml logs --tail=100 caddy
```
