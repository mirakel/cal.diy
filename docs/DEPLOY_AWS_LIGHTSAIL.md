# Deploy Cal.diy API v2 to AWS Lightsail with Docker Compose + GitHub Actions

> Read this document in: **English** | [Español](./DEPLOY_AWS_LIGHTSAIL.es.md)

Production-oriented, from-scratch guide to deploy the **Cal.diy (Cal.com DIY) API v2**
to a single **AWS Lightsail** Ubuntu instance. GitHub Actions builds and pushes the
Docker image, then connects over SSH and asks **Docker Compose** to recreate the
`calcom-api` service. PostgreSQL lives outside the instance (Neon); Redis runs as a
compose service.

This guide assumes **nothing is configured yet**: no Lightsail instance, no firewall
rules, no GitHub secrets and no workflow file in the new repository
(`https://github.com/kennypowers1987/hillharbor_calendar`).

---

## 0. Placeholders and prerequisites

Replace every placeholder before running any command.

| Placeholder | Meaning | Example (this environment) |
| --- | --- | --- |
| `<STATIC_IP>` | Lightsail Static IP | `3.128.79.75` |
| `<PROJECT_DIR>` | Directory holding `.env` + `docker-compose.yml` | `/home/ubuntu/cal-diy` |
| `<DOCKERHUB_USER>` | Docker Hub account | `mirakel14` |
| `<IMAGE_NAME>` | API image name | `cal-diy-api` |
| `<REPO_URL>` | GitHub repository | `https://github.com/kennypowers1987/hillharbor_calendar` |
| `<NEON_HOST>` | Neon pooled/direct host | `<project>-pooler.<region>.aws.neon.tech` |
| `<SSH_KEY_PATH>` | Local path to the `.pem` file | `~/.ssh/lightsail.pem` |

Prerequisites:

- AWS account with Lightsail access.
- Docker Hub account.
- Neon (or any managed PostgreSQL 15+) project.
- A local copy of this repository with `Dockerfile.api` and `docker-compose-aws.yml`.
- Optional: a domain name (only needed for the TLS appendix).

---

## 1. Architecture overview

```
Developer push to main
        │
        ▼
GitHub Actions (.github/workflows/deploy-api.yml)
        │  1. docker/build-push-action (Dockerfile.api)
        │  2. docker/login-action
        ▼
Docker Hub: <DOCKERHUB_USER>/<IMAGE_NAME>:latest
        │
        ▼  appleboy/ssh-action (port 22)
AWS Lightsail (Ubuntu)
        cd <PROJECT_DIR>
        docker compose pull calcom-api
        docker compose up -d --no-deps --force-recreate calcom-api
        │
        ├── calcom-api   :3000  image <DOCKERHUB_USER>/<IMAGE_NAME>:latest
        ├── calcom (web) :3001  image <DOCKERHUB_USER>/cal-diy:latest
        ├── redis        6379 inside the "stack" network
        └── database     local Postgres, NOT used when DATABASE_URL points to Neon
        │
        └── External: Neon PostgreSQL (TLS)
```

Port map on a single instance:

| Port | Service | Exposure |
| --- | --- | --- |
| `22` | SSH (GitHub Actions + you) | Restrict to your IP + GitHub runner ranges |
| `3000` | API v2 (`calcom-api`) | Public (put a TLS proxy in front for real production) |
| `3001` | Web app (`calcom`) | Public |
| `6379` | Redis | **Should stay internal**. The reference compose publishes it; remove that rule (see §4.6) |

Memory budget on a 4 GB / 2 vCPU / 80 GB SSD instance:

| Component | Approx. RAM |
| --- | --- |
| Ubuntu + Docker daemon | 400–600 MB |
| `calcom-api` (NestJS + slots workers) | 1.2–1.8 GB |
| `calcom` web app (Next.js) | 800 MB–1 GB |
| `redis` | 50–150 MB |
| `database` (local Postgres, idle) | ~100–300 MB |
| Headroom | ~300–500 MB |

> The Docker build runs on GitHub-hosted runners, not on Lightsail. The 2 vCPUs are
> only used by `docker compose pull` and the container runtime.
>
> Compose v1 (`docker-compose`, Python) **ignores `deploy.resources.limits`**, so the
> memory caps in the reference compose file are decorative. Use Compose v2 (§3.1).

---

## 2. Create and configure the AWS Lightsail instance

### 2.1 Create the instance

1. Open the [Lightsail console](https://lightsail.aws.amazon.com/) in your target region.
2. **Create instance** → **Linux/Unix** → **OS Only** → **Ubuntu 22.04 LTS** (or 24.04).
3. Plan: **at least 2 GB RAM**; recommended **4 GB RAM / 2 vCPU / 80 GB SSD** (US$24/mo tier).
4. Name it (e.g. `cal-diy-prod`) and create it.

Wait until the instance state is **Running** and note the public IPv4 address.

### 2.2 Attach a Static IP

1. Instance → **Networking** tab → **Create static IP**.
2. Attach it to the instance.
3. Use that IP everywhere this guide says `<STATIC_IP>`. Example: `3.128.79.75`.

> Without a Static IP the address changes on every stop/start and all your GitHub
> secrets and URLs break.

### 2.3 Open the firewall (IPv4 Firewall)

Instance → **Networking** → **IPv4 Firewall** → **Add rule**:

| Application | Protocol | Port | Restricted to |
| --- | --- | --- | --- |
| SSH | TCP | `22` | Your IP (and, if possible, GitHub Actions IP ranges) |
| Custom | TCP | `3000` | Anywhere (`0.0.0.0/0`) |
| Custom | TCP | `3001` | Anywhere (`0.0.0.0/0`) |

> Never open `6379` in the Lightsail firewall. Redis must stay inside the Docker
> network.
> Binding SSH to `0.0.0.0/0` is a brute-force magnet; restrict it.

### 2.4 Download the SSH key

1. Account → **SSH keys** → download the default key for the region (`.pem`).
2. Move it to a safe location and lock its permissions:

```bash
mv ~/Downloads/LightsailDefaultKey-*.pem <SSH_KEY_PATH>
chmod 400 <SSH_KEY_PATH>
```

Test the connection (replace `<STATIC_IP>`):

```bash
ssh -i <SSH_KEY_PATH> ubuntu@<STATIC_IP>
```

---

## 3. Prepare the Ubuntu server (Docker and networking)

### 3.1 Install Docker Engine + Compose v2

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

docker --version
docker compose version
```

> Do **not** use Ubuntu's `docker.io` + `docker-compose` packages. Compose v1 is
> end-of-life, requires Python, and silently ignores `deploy.resources.limits`
> (your `mem` caps would never apply). If you already installed them:
>
> ```bash
> sudo apt-get remove -y docker-compose
> ```
>
> and keep using `docker compose` (with a space) from now on.

### 3.2 Allow `ubuntu` to run Docker without sudo

GitHub Actions connects as `ubuntu`; without this step every `docker` command in the
workflow fails with `permission denied while trying to connect to the Docker daemon`.

```bash
sudo usermod -aG docker ubuntu
```

Then apply the group. Either log out and back in, or:

```bash
newgrp docker
docker ps
```

### 3.3 Docker log rotation (protects the 80 GB SSD)

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF
sudo systemctl restart docker
```

### 3.4 Add 2 GB of swap (4 GB instances only)

Without swap a traffic spike or a container restart can trigger the OOM killer and
take Redis down with the API.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
free -h
```

### 3.5 Bring up the stack with Docker Compose

**Prerequisite:** the two files described in §4.1 must exist in `<PROJECT_DIR>`:

```
<PROJECT_DIR>/.env
<PROJECT_DIR>/docker-compose.yml
```

Then:

```bash
cd <PROJECT_DIR>
docker compose config --quiet && echo "compose OK"
docker compose up -d
docker compose ps
```

Expected containers: `calcom-api`, `calcom`, `redis`, `database`.

The compose file declares a network named `stack`; all services join it, so
`REDIS_URL=redis://redis:6379` resolves by service/container name. You do **not**
need to create any network or Redis container by hand — Compose does it.

Verify Redis from inside the API container:

```bash
docker compose exec calcom-api sh -c 'echo > /dev/tcp/redis/6379 && echo redis-reachable'
docker compose exec redis redis-cli ping   # PONG
```

> **Alternative without Compose (not recommended if Compose is already running).**
> `docker run` and Compose must not manage the same container name. After
> `docker compose down`, you could run:
>
> ```bash
> docker network create calcom-net
> docker volume create redis-data
> docker run -d --name calcom-redis --network calcom-net \
>   --network-alias redis --restart unless-stopped \
>   -v redis-data:/data redis:7-alpine
> docker run -d --name calcom-api --network calcom-net \
>   --restart unless-stopped -p 3000:3000 \
>   --env-file <PROJECT_DIR>/.env <DOCKERHUB_USER>/<IMAGE_NAME>:latest
> ```
>
> With that model, `REDIS_URL` must still resolve to `redis`, hence the
> `--network-alias redis`, and the memory limits must be added manually.

---

## 4. Server environment variables

### 4.1 Files required on the instance

Only **two files** are uploaded, both into `<PROJECT_DIR>` (`/home/ubuntu/cal-diy`):

| File on the instance | Content | Source in the repository |
| --- | --- | --- |
| `<PROJECT_DIR>/.env` | Real environment values, following the structure of `.env.aws.example` | `.env.aws.example` (filled in) |
| `<PROJECT_DIR>/docker-compose.yml` | Services `calcom-api`, `calcom`, `redis` (+ unused `database`) | `docker-compose-aws.yml`, **uploaded renamed** |

> The compose file **must** be named `docker-compose.yml` on the server, otherwise
> `docker compose up -d` will not find it and will fail with
> `no configuration file provided`.
>
> Do not upload `.env.aws` under its original name: the compose file reads `.env`.

```bash
# Create the project directory once
ssh -i <SSH_KEY_PATH> ubuntu@<STATIC_IP> 'mkdir -p <PROJECT_DIR>'

# .env (real values)
scp -i <SSH_KEY_PATH> ./.env.aws ubuntu@<STATIC_IP>:<PROJECT_DIR>/.env

# compose file, renamed on the server
scp -i <SSH_KEY_PATH> ./docker-compose-aws.yml ubuntu@<STATIC_IP>:<PROJECT_DIR>/docker-compose.yml

# lock the secrets file down
ssh -i <SSH_KEY_PATH> ubuntu@<STATIC_IP> 'chmod 600 <PROJECT_DIR>/.env'

# check
ssh -i <SSH_KEY_PATH> ubuntu@<STATIC_IP> 'ls -l <PROJECT_DIR>'
```

Expected listing: `.env` (`-rw-------`) and `docker-compose.yml`.

`chmod 600` matters: Docker reads the file to inject the environment.

### 4.2 Template (mirrors `.env.aws.example`)

```dotenv
# Compose project name
COMPOSE_PROJECT_NAME=cal-diy

# Runtime
NODE_ENV=development
API_PORT=3000

# Public URLs
NEXT_PUBLIC_WEBAPP_URL=http://<STATIC_IP>:3001
WEB_APP_URL=http://<STATIC_IP>:3001
NEXT_PUBLIC_API_V2_URL=http://<STATIC_IP>:3000/api/v2
API_URL=http://<STATIC_IP>:3000

# Database (Neon)
DATABASE_URL=postgresql://USER:PASSWORD@<NEON_POOLED_HOST>/<DATABASE>?sslmode=require
DATABASE_DIRECT_URL=postgresql://USER:PASSWORD@<NEON_DIRECT_HOST>/<DATABASE>?sslmode=require
DATABASE_READ_URL=postgresql://USER:PASSWORD@<NEON_POOLED_HOST>/<DATABASE>?sslmode=require
DATABASE_WRITE_URL=postgresql://USER:PASSWORD@<NEON_POOLED_HOST>/<DATABASE>?sslmode=require

# Redis (compose service)
REDIS_URL=redis://redis:6379

# Secrets
NEXTAUTH_SECRET=<GENERATE_ME>
JWT_SECRET=<GENERATE_ME>
CALENDSO_ENCRYPTION_KEY=<GENERATE_ME>

# Cal.com
CALCOM_TELEMETRY_DISABLED=true
NEXT_PUBLIC_LICENSE_CONSENT=true
API_KEY_PREFIX=cal_
LOG_LEVEL=info
ENABLE_SLOTS_WORKERS=true

# Stripe (optional)
STRIPE_API_KEY=
STRIPE_WEBHOOK_SECRET=

# SMTP
EMAIL_SERVER_HOST=smtp.example.com
EMAIL_SERVER_PORT=587
EMAIL_SERVER_USER=notifications@example.com
EMAIL_SERVER_PASSWORD=<SMTP_APP_PASSWORD>
EMAIL_FROM=Cal.diy <notifications@example.com>

# Node
NODE_OPTIONS=--max-old-space-size=2048
WEB_CONCURRENCY=1

# TLS / proxy
SSL_ENABLED=false
NODE_TLS_REJECT_UNAUTHORIZED=0
EXPRESS_TRUST_PROXY=false
NO_HTTPS=true

# Swagger
SWAGGER_CUSTOM_SITE_TITLE=Cal.com API
SWAGGER_CUSTOM_CSS_URL=/docs/swagger-ui.css
SWAGGER_CUSTOM_JS=/docs/swagger-ui-bundle.js
```

The compose file already injects `API_PORT=3000`, `DATABASE_READ_URL` and
`DATABASE_WRITE_URL` from `DATABASE_URL`, but keeping them in `.env` makes the file
usable outside Compose too.

### 4.3 How to obtain or generate each value

| Variable | How to obtain / generate it |
| --- | --- |
| `COMPOSE_PROJECT_NAME` | `cal-diy`. Keeps container/volume prefixes stable regardless of directory name. |
| `NODE_ENV` | `development`. The API only registers Swagger (`/v2/docs`) when this is not `production` (`apps/api/v2/src/main.ts:69-71`). With `production`, `apps/api/v2/src/app.module.ts:38` appends `?tls=true` to `REDIS_URL` and the plain local Redis fails the TLS handshake. |
| `API_PORT` | `3000`. **The app ignores `PORT`**; it reads `API_PORT`, default `5555` (`apps/api/v2/src/config/app.ts:6`). Without it the container listens on 5555 while you publish 3000. The compose file already sets it to `3000`. |
| `NEXT_PUBLIC_WEBAPP_URL`, `WEB_APP_URL` | `http://<STATIC_IP>:3001` (web app origin). |
| `NEXT_PUBLIC_API_V2_URL` | `http://<STATIC_IP>:3000/api/v2`. |
| `API_URL` | `http://<STATIC_IP>:3000`. |
| `DATABASE_URL` | Neon → Project → **Connect** → copy the **Pooled** connection string. Keep `?sslmode=require`. |
| `DATABASE_DIRECT_URL` | Same as above but the **Direct** (non-pooler) host. Used for migrations. |
| `DATABASE_READ_URL`, `DATABASE_WRITE_URL` | Not in the original `.env.aws`, but **required**: `apps/api/v2/src/config/app.ts:26-27` reads them without a fallback and the process refuses to boot. Use the pooled URL for both. |
| `POSTGRES_*` | Legacy, only for the local Postgres service in the compose file. Ignore when using Neon. |
| `REDIS_URL` | `redis://redis:6379` (matches the compose service/container name `redis`). |
| `NEXTAUTH_SECRET` | `openssl rand -base64 32`. Must match the web app value. |
| `JWT_SECRET` | `openssl rand -base64 32`. |
| `CALENDSO_ENCRYPTION_KEY` | `openssl rand -base64 24` → exactly 32 characters. Losing it makes stored calendar credentials undecryptable. |
| `CALCOM_TELEMETRY_DISABLED` | `true` for self-hosted. |
| `NEXT_PUBLIC_LICENSE_CONSENT` | `true`. |
| `API_KEY_PREFIX` | `cal_` (prefix of generated API keys). |
| `LOG_LEVEL` | `info` (or `debug` while troubleshooting). |
| `ENABLE_SLOTS_WORKERS` | `true`; enables `SlotsWorkerService_2024_04_15`. |
| `STRIPE_*` | Stripe Dashboard → Developers → API keys / Webhooks. Leave empty if unused. |
| `EMAIL_SERVER_*` | Your SMTP provider. For Gmail use an **App Password**, never the account password. |
| `NODE_OPTIONS` | `--max-old-space-size=2048` on a 4 GB host shared with the web app. |
| `WEB_CONCURRENCY` | `1` on 2 vCPUs. |
| `NODE_TLS_REJECT_UNAUTHORIZED` | `0` disables TLS verification globally (MITM risk). Acceptable only for throwaway environments. |
| `EXPRESS_TRUST_PROXY`, `NO_HTTPS`, `SSL_ENABLED` | `false` / `true` / `false` when terminating TLS at a proxy. |
| `SWAGGER_*` | Only cosmetic for the Swagger UI. |

### 4.4 `.env` rules

Docker Compose parses quoted values correctly (it strips the quotes), but
`docker run --env-file` does **not**, so keep the safe style anyway:

```dotenv
# RISKY outside Compose - quotes become part of the value
# -> Node: "invalid value for NODE_OPTIONS"
NODE_OPTIONS="--max-old-space-size=2048"

# WRONG - `export` is not supported; the key becomes "export NODE_ENV"
export NODE_ENV=production

# WRONG - everything after # may be truncated; URL-encode values containing #
DATABASE_URL=postgresql://user:pa#ss@host/db

# CORRECT
NODE_OPTIONS=--max-old-space-size=2048
NODE_ENV=development
DATABASE_URL=postgresql://user:p%23ss@host/db
```

Values with spaces (like `EMAIL_FROM=Cal.diy <a@b.c>`) are valid; trailing spaces
are passed through as-is.

Validate the file without starting the stack:

```bash
cd <PROJECT_DIR>
docker compose config --quiet && echo "compose + .env OK"
docker compose run --rm --no-deps calcom-api printenv API_PORT
```

### 4.5 Security warning: secrets inside the image

`Dockerfile.api` runs `COPY . .`. If `.env.aws` is present in the build context it
ends up **inside the public image** and anyone can extract it with
`docker history`/`docker save`. `.dockerignore` in this repository must contain:

```
.env
.env.*
**/.env
**/.env.*
```

If a previous image was published with secrets inside, rotate every credential
(Neon password, `NEXTAUTH_SECRET`, `JWT_SECRET`, `CALENDSO_ENCRYPTION_KEY`,
`STRIPE_*`, SMTP password) and delete the old tags from Docker Hub.

Also: the `build-args` in the workflow (`DATABASE_URL`, `NEXTAUTH_SECRET`, ...) are
**inert** because `Dockerfile.api` declares no `ARG`. They do not leak, but they give
a false sense of security. All secrets belong in the runtime `.env`.

### 4.6 Advisory notes about the reference compose file

These are **warnings only**; apply them at your own pace:

1. **Redis is published.** `redis` has `ports: ["6379:6379"]` in
   `docker-compose-aws.yml`. That exposes an unauthenticated Redis to the internet.
   Remove the `ports` block (Compose networking is enough). If you already opened
   `6379` in Lightsail, close it.
2. **`database` is dead weight.** The `calcom`/`calcom-api` services use
   `DATABASE_URL` from `.env` (Neon), so the local Postgres container consumes RAM
   for nothing. Removing it also means removing the `depends_on: database` entries.
3. **`deploy.resources.limits` requires Compose v2.** With `docker-compose` v1 the
   memory caps are ignored; with the plugin they are enforced. Alternatively use
   `mem_limit: 1500m` in `calcom-api` and `256m` in `redis`.
4. **`version: '3.8'` is obsolete** in Compose v2 (warning on every command). Delete
   the line.
5. **`NODE_TLS_REJECT_UNAUTHORIZED=0`** disables certificate validation for the whole
   process. Keep it only while the environment is disposable.

---

## 5. GitHub Actions secrets

Repository → **Settings** → **Secrets and variables** → **Actions** →
**New repository secret**.

`<REPO_URL>/settings/secrets/actions`

| Secret | Value | How to obtain it |
| --- | --- | --- |
| `DOCKERHUB_USERNAME` | Docker Hub user | Your Docker Hub account name. |
| `DOCKERHUB_TOKEN` | Docker Hub access token | Docker Hub → Account settings → Personal access tokens → **Read & Write**. Never use your password. |
| `PRODUCTION_DATABASE_URL` | Neon pooled URL | Only used as a build arg (inert). Kept for compatibility. |
| `NEXTAUTH_SECRET` | `openssl rand -base64 32` | Same value as in the server `.env`. |
| `CALENDSO_ENCRYPTION_KEY` | `openssl rand -base64 24` | Same value as in the server `.env`. |
| `LIGHTSAIL_HOST` | `<STATIC_IP>` | The Lightsail Static IP. |
| `LIGHTSAIL_USERNAME` | `ubuntu` | Default user of the Ubuntu Lightsail image. |
| `LIGHTSAIL_SSH_KEY` | Full `.pem` contents | `cat <SSH_KEY_PATH>` including the `-----BEGIN ...-----` and `-----END ...-----` lines. |
| `LIGHTSAIL_API_URL` | `http://<STATIC_IP>:3000` | API public URL. |
| `LIGHTSAIL_WEB_URL` | `http://<STATIC_IP>:3001` | Web app public URL, injected as `NEXT_PUBLIC_WEBAPP_URL` at build time. |

> The image is **public**, so the server does not need `docker login`. If you switch
> the Docker Hub repository to private, add a `docker login` step inside the SSH
> script.

> If the repository remote contains a Personal Access Token in its URL
> (`git remote -v`), rotate it and rewrite the remote:
> `git remote set-url origin <REPO_URL>`.

---

## 6. Workflow file `.github/workflows/deploy-api.yml`

Create the file from scratch in the new repository:

```yaml
name: Build, Push & Deploy API to AWS Lightsail

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and Push API Image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.api
          push: true
          tags: |
            <DOCKERHUB_USER>/<IMAGE_NAME>:latest

          cache-from: type=gha
          cache-to: type=gha,mode=max

          # These build args are inert: Dockerfile.api declares no ARG.
          # Runtime secrets belong in <PROJECT_DIR>/.env on the server.
          build-args: |
            NEXT_PUBLIC_WEBAPP_URL=${{ secrets.LIGHTSAIL_WEB_URL }}
            MAX_OLD_SPACE_SIZE=4096

      - name: Deploy to AWS Lightsail
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.LIGHTSAIL_HOST }}
          username: ${{ secrets.LIGHTSAIL_USERNAME }}
          key: ${{ secrets.LIGHTSAIL_SSH_KEY }}
          port: 22
          script_stop: true
          script: |
            cd <PROJECT_DIR>
            docker compose pull calcom-api
            docker compose up -d --no-deps --force-recreate calcom-api
            docker image prune -f
```

Notes:

- `cache-from/cache-to: type=gha` reuses layers between runs through the GitHub
  Actions cache.
- `script_stop: true` makes the remote script abort on the first non-zero exit.
- `docker compose pull calcom-api` downloads the new `:latest` image; the following
  `up -d --no-deps --force-recreate calcom-api` recreates **only** the API container.
  The web app, Redis and their data are untouched.
- The remote user must be in the `docker` group (§3.2) and the compose file must be
  named `docker-compose.yml` in `<PROJECT_DIR>` (§4.1).
- `NEXT_PUBLIC_*` values are baked at **build time**; everything else is read at
  **runtime** from the `.env`.

---

## 7. Verification and troubleshooting

### 7.1 Container health

```bash
cd <PROJECT_DIR>
docker compose ps
docker compose logs -f calcom-api
```

Healthy startup shows the NestJS bootstrap log, Prisma connecting, and the slots
worker (`SlotsWorkerService_2024_04_15`). Redis connectivity:

```bash
docker compose exec redis redis-cli ping                                    # PONG
docker compose exec calcom-api sh -c 'echo > /dev/tcp/redis/6379 && echo redis-reachable'
```

To confirm which compose file the running stack came from:

```bash
docker inspect calcom-api \
  --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}'
```

### 7.2 Endpoints

```bash
curl -i http://<STATIC_IP>:3000/v2/docs   # Swagger UI (only with NODE_ENV=development)
curl -i http://<STATIC_IP>:3000/          # 404 is EXPECTED on the API v2 root
```

`/v2/docs` returns 404 when `NODE_ENV=production`; that is a code-level condition
(`apps/api/v2/src/main.ts:69-71`), not a deployment failure.

### 7.3 Common errors

| Symptom | Cause | Fix |
| --- | --- | --- |
| `no configuration file provided` | `docker compose` run outside `<PROJECT_DIR>` or file named `docker-compose-aws.yml` | `cd <PROJECT_DIR>`; rename the file to `docker-compose.yml` |
| `network stack not found` | Compose created the stack under another project/file | `docker compose up -d` from `<PROJECT_DIR>`; never mix with `docker run` |
| `Error: connect ECONNREFUSED 6379` / `getaddrinfo ENOTFOUND redis` | API container not on the `stack` network, or Redis container is down | `docker compose up -d redis`, `docker compose ps` |
| `ERR_SSL_WRONG_VERSION_NUMBER` / TLS handshake on Redis | `NODE_ENV=production` appends `?tls=true` to a non-TLS Redis | Use `NODE_ENV=development`, or a TLS-enabled Redis |
| `Missing environment variable: DATABASE_READ_URL` | Variable absent (no fallback in `config/app.ts`) | Add `DATABASE_READ_URL` and `DATABASE_WRITE_URL` (or rely on the compose overrides) |
| Container up but `ERR_CONNECTION_REFUSED` on `:3000` | App listening on 5555 because `API_PORT` is unset | Ensure `API_PORT=3000` in `.env` or in the compose `environment` block |
| `invalid value for NODE_OPTIONS` | Quotes and you switched to `docker run --env-file` | Remove the quotes |
| `permission denied` reading `.env` | Wrong owner/mode | `chmod 600 <PROJECT_DIR>/.env` |
| Workflow: `permission denied ... docker.sock` | `ubuntu` not in `docker` group | `sudo usermod -aG docker ubuntu`, re-login |
| `Bind for 0.0.0.0:3000 failed: port is already allocated` | Container created manually with `docker run` | `docker rm -f calcom-api` and use Compose only |
| `container name "/calcom-api" is already in use` | Leftover `docker run` container | `docker rm -f calcom-api`, then `docker compose up -d` |
| Container `OOMKilled` | Memory pressure | Lower `NODE_OPTIONS` to 2048, add swap, use Compose v2 limits |
| `no space left on device` | Image layers + logs | `docker system prune -af --filter until=168h` |
| SSH action: `ssh: handshake failed` | Wrong key or firewall | Verify `ssh -i <SSH_KEY_PATH> ubuntu@<STATIC_IP>` locally; check port 22 rule |

### 7.4 Useful maintenance commands

```bash
docker compose ps
docker stats --no-stream
docker image prune -af --filter until=168h
journalctl -u docker --no-pager -n 100
df -h
```

---

## Appendix A — HTTPS with Caddy (optional)

Closing `3000`/`3001` and terminating TLS in Caddy is strongly recommended for any
environment that is not a throwaway test.

```bash
sudo apt-get install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' \
  | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' \
  | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt-get update && sudo apt-get install -y caddy
```

`/etc/caddy/Caddyfile`:

```caddyfile
api.example.com {
  reverse_proxy 127.0.0.1:3000
}

app.example.com {
  reverse_proxy 127.0.0.1:3001
}
```

```bash
sudo systemctl reload caddy
```

Then update `NEXT_PUBLIC_WEBAPP_URL`, `WEB_APP_URL`, `API_URL`,
`NEXT_PUBLIC_API_V2_URL` to the HTTPS domains, set `NO_HTTPS=false`,
`EXPRESS_TRUST_PROXY=true`, and remove `NODE_TLS_REJECT_UNAUTHORIZED=0`.
Finally close ports `3000` and `3001` in the Lightsail firewall and keep only
`80`/`443`.

---

## Appendix B — Deploying the web app on port 3001

The API workflow only recreates `calcom-api`. The web app is a separate service in
the same compose file. To release it, either run on the server:

```bash
cd <PROJECT_DIR>
docker compose pull calcom
docker compose up -d --no-deps --force-recreate calcom
```

or create `.github/workflows/deploy-web.yml` with the same secrets and:

```yaml
          script: |
            cd <PROJECT_DIR>
            docker compose pull calcom
            docker compose up -d --no-deps --force-recreate calcom
            docker image prune -f
```

The web container must share `NEXTAUTH_SECRET`, `CALENDSO_ENCRYPTION_KEY` and
`DATABASE_URL` with the API environment (it already does, through `env_file: .env`).

---

> Read this document in: **English** | [Español](./DEPLOY_AWS_LIGHTSAIL.es.md)
