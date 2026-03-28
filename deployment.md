# LionTech CRM — Deployment Guide

A new engineer should be able to deploy the CRM from scratch using only this guide.

***

## Architecture Overview

| Component          | Technology                       | Hosting                         |
| ------------------ | -------------------------------- | ------------------------------- |
| Backend API        | Node.js 20 / Express             | Railway (`api` service)         |
| Frontend           | React / Vite → nginx             | Railway (`frontend` service)    |
| Database           | PostgreSQL 16                    | Railway managed Postgres plugin |
| Container registry | GitHub Container Registry (GHCR) | CI/CD pipeline                  |
| CI/CD              | GitHub Actions                   | `.github/workflows/ci.yml`      |

***

## Local Development

### Prerequisites

* Docker Desktop ≥ 4.x
* Node.js 20 (optional — only needed to run scripts outside Docker)
* `jq` (optional — useful for API testing)

### Quick start

```bash
# Clone the repo and enter the project directory
git clone <repo-url>
cd <repo-dir>

# Build images and start all services
docker compose up --build
```

Services available:

| Service     | URL                   |
| ----------- | --------------------- |
| Backend API | http://localhost:3000 |
| Frontend    | http://localhost:5173 |
| PostgreSQL  | localhost:5432        |

Migrations run automatically via the `command: node src/server.js` startup path — the migration runner is called explicitly via `npm run migrate` if you need to apply them manually (see below).

### Run migrations manually

```bash
docker compose exec backend npm run migrate
```

The migration runner (`src/migrations/run.js`) reads `.sql` files from `src/migrations/` in filename order and skips any already applied version. It is idempotent.

> **Note:** There is no `migrate:down` or rollback script. Rollback is handled at the Railway deployment level (see Rollback Procedure below). Write backward-compatible migrations.

### Stop services

```bash
docker compose down          # stop containers, keep postgres volume
docker compose down -v       # stop + delete postgres volume (full reset)
```

### Environment variables (local)

Docker Compose injects all required variables inline — no `.env` file is needed for local development. The values in `docker-compose.yml` are intentionally insecure dev defaults. **Never use them in production.**

***

## CI Pipeline (GitHub Actions)

File: `.github/workflows/ci.yml`

| Stage        | Trigger           | What runs                                                          |
| ------------ | ----------------- | ------------------------------------------------------------------ |
| Lint         | Every PR and push | ESLint + TypeScript type-check (frontend)                          |
| Test         | After lint        | Jest unit + integration tests (backend runs against real Postgres) |
| Build & Push | After tests       | Multi-stage Docker build → push to GHCR                            |
| Deploy       | `main` push only  | `railway up` + pre-deploy migration hook + health check            |

### Required GitHub Secrets

| Secret          | Description                                     |
| --------------- | ----------------------------------------------- |
| `RAILWAY_TOKEN` | Railway API token — Project → Settings → Tokens |

### Required GitHub Variables

| Variable          | Description                                  |
| ----------------- | -------------------------------------------- |
| `VITE_API_URL`    | Public URL of the Railway `api` service      |
| `RAILWAY_APP_URL` | Public URL of the Railway `frontend` service |
| `RAILWAY_API_URL` | Same as `VITE_API_URL` (used in smoke test)  |

***

## Railway Setup (First-time)

### 1. Create project

1. Log in to [railway.app](https://railway.app)
2. New Project → "Empty Project" → name it `crm-platform`

### 2. Add PostgreSQL

New Service → Database → PostgreSQL. Railway auto-injects `DATABASE_URL` into other services in the same project.

### 3. Add Backend service

1. New Service → GitHub Repo → select this repo
2. Set **Root Directory**: `crm-backend`
3. Set **Dockerfile Path**: `Dockerfile`
4. Add environment variables (table below)

Railway also reads `railway.toml` from the repo root. The `[services.deploy.preDeployCommand]` hook runs `npm run migrate` before traffic is cut over — migrations are always applied before the new container goes live.

### 4. Add Frontend service

1. New Service → GitHub Repo → select this repo
2. Set **Root Directory**: `crm-frontend`
3. Set **Dockerfile Path**: `Dockerfile`
4. Add `VITE_API_URL` \= `https://<backend-service-url>`

### Backend environment variables (Railway)

Set these in Railway → Service → Variables. **Never hardcode in files or commit to git.**

| Variable                 | Value / Notes                                                   |
| ------------------------ | --------------------------------------------------------------- |
| `NODE_ENV`               | `production`                                                    |
| `PORT`                   | Auto-injected by Railway; Express reads `process.env.PORT`      |
| `DATABASE_URL`           | Auto-injected by Railway Postgres plugin                        |
| `JWT_SECRET`             | `openssl rand -base64 64`                                       |
| `JWT_EXPIRES_IN`         | `15m`                                                           |
| `JWT_REFRESH_SECRET`     | `openssl rand -base64 64` (must differ from `JWT_SECRET`)       |
| `JWT_REFRESH_EXPIRES_IN` | `7d`                                                            |
| `CORS_ORIGIN`            | `https://<frontend-service-url>` (comma-separated for multiple) |
| `RATE_LIMIT_WINDOW_MS`   | `900000`                                                        |
| `RATE_LIMIT_MAX`         | `100`                                                           |
| `DB_POOL_MIN`            | `2`                                                             |
| `DB_POOL_MAX`            | `10`                                                            |
| `LOG_LEVEL`              | `info`                                                          |

***

## Database Migrations

Migrations are applied automatically by the Railway pre-deploy hook (`npm run migrate`) before traffic switches to the new container.

```toml
# railway.toml (already committed)
[services.deploy.preDeployCommand]
command = "npm run migrate"
```

**Manual migration** (emergency or first deploy):

```bash
# Via Railway CLI
railway run --service api -- npm run migrate

# Via docker compose (local)
docker compose exec backend npm run migrate
```

**Schema rollback:** There is no automated down-migration. If a migration introduces a breaking change, redeploy the previous image (see Rollback Procedure) and fix the migration SQL before re-deploying.

***

## Zero-Downtime Deploy Strategy

Railway uses rolling deploys:

1. New container is built and health-checked at `GET /health`
2. Pre-deploy migration hook runs (`npm run migrate`)
3. Once the new container is healthy, traffic is switched over
4. Old container is stopped

**Golden rule:** Write migrations that are backward-compatible with the currently-running code — new nullable columns, new tables, no dropped columns mid-deploy.

***

## Health Checks

| Service  | Endpoint      | Expected response                                        |
| -------- | ------------- | -------------------------------------------------------- |
| Backend  | `GET /health` | `200 { "status": "ok", "db": "ok", "timestamp": "..." }` |
| Frontend | `GET /health` | `200 ok` (served by nginx)                               |

`db` is `"degraded"` if PostgreSQL is unreachable; the endpoint returns `503` in that case.

***

## Rollback Procedure

### Automatic (Railway dashboard)

Railway keeps the last 3 successful deployments:

1. Railway Dashboard → Service → Deployments
2. Click the previous successful deployment → "Redeploy"

### Manual (CLI)

```bash
# List recent deployments
railway deployments list

# Redeploy a specific deployment
railway deployments redeploy <deployment-id>
```

### Database rollback

If a migration must be reversed:

1. Write a compensating SQL migration (e.g. `002_rollback_xxx.sql`) and merge it
2. Or redeploy the previous image and manually run compensating SQL via:

```bash
railway run --service api -- node -e "require('./src/config/database').query('...')"
```

***

## Secrets Management

* **Never** commit secrets to git
* All secrets live in Railway environment variables
* Local dev uses the insecure defaults in `docker-compose.yml` (gitignored `.env` not required)
* JWT secrets should be rotated every 90 days; old tokens expire naturally within 15 minutes (access) / 7 days (refresh)
* CI uses GitHub Secrets only for `RAILWAY_TOKEN`

***

## Monitoring & Logs

```bash
# Tail logs
railway logs --service api --tail 100
railway logs --service frontend --tail 50

# Check deployment status
railway status
```

Configure Railway webhook alerts (Project → Settings → Notifications) to post to Slack on deploy failure or health check failure.

***

## Troubleshooting

| Symptom                                    | Likely cause                         | Fix                                                                     |
| ------------------------------------------ | ------------------------------------ | ----------------------------------------------------------------------- |
| Deploy fails at migration step             | Bad migration SQL                    | Fix the SQL, push new commit                                            |
| Health check returns 503                   | App crashed on startup               | `railway logs --service api`                                            |
| Frontend blank page                        | `VITE_API_URL` wrong or CORS blocked | Verify env vars + `CORS_ORIGIN` on backend                              |
| Database connection refused                | `DATABASE_URL` not set               | Ensure Postgres plugin is linked to the service                         |
| 401 on all API calls after redeploy        | `JWT_SECRET` changed                 | Verify env var is unchanged; existing tokens will re-auth on next login |
| `docker compose up` fails on port conflict | Port 3000 or 5173 already in use     | Stop conflicting processes or change host port mappings                 |
