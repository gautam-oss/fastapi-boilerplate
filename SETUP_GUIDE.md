# FastAPI Boilerplate — Complete Setup Guide
> **Real-time setup experience** captured on this device (Windows 11, May 2026).  
> Every command here was actually run and verified on this machine.

---

## Table of Contents
1. [What We're Setting Up](#1-what-were-setting-up)
2. [Prerequisites (Zero to Ready)](#2-prerequisites-zero-to-ready)
3. [Project Structure Overview](#3-project-structure-overview)
4. [Environment Files Explained](#4-environment-files-explained)
5. [Local Environment](#5-local-environment-uvicorn--hot-reload)
6. [Staging Environment](#6-staging-environment-gunicorn--4-workers)
7. [Production Environment](#7-production-environment-gunicorn--nginx)
8. [Post-Startup Steps](#8-post-startup-steps-every-environment)
9. [Switching Between Environments](#9-switching-between-environments)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. What We're Setting Up

This boilerplate ships **three deployment modes**:

| Mode | Server Stack | Use For | Port |
|------|-------------|---------|------|
| **Local** | Uvicorn (1 process, hot-reload) | Day-to-day development | 8000 |
| **Staging** | Gunicorn + 4 Uvicorn workers | QA / pre-prod testing | 8000 |
| **Production** | Gunicorn + 4 workers + Nginx | Live traffic | 80 |

All three use the **same Docker Compose pattern**:

```
web (FastAPI app)
├── db  (PostgreSQL 13)
├── redis  (Alpine Redis)
└── worker  (ARQ background jobs)
```

Production adds `nginx` as a reverse proxy in front of `web`.

---

## 2. Prerequisites (Zero to Ready)

### Device State at Start
- Windows 11, fresh install
- No Python, no Docker, no WSL2

### Step 1 — Enable WSL2
Docker Desktop requires WSL2 on Windows. Run in PowerShell **as Administrator**:

```powershell
wsl --install --no-distribution
```

> **What happened on this device:** WSL2 kernel installed successfully. Output confirmed:
> `"Windows Subsystem for Linux has been installed."`
> `"The requested operation is successful. Changes will not be effective until the system is rebooted."`

⚠️ **You must reboot after this step.** WSL2 won't be active until you do.

```powershell
Restart-Computer
```

### Step 2 — Install Docker Desktop
After the reboot, run in PowerShell:

```powershell
winget install --id Docker.DockerDesktop --silent --accept-package-agreements --accept-source-agreements
```

> **What happened on this device:** Installed successfully via winget.
> Docker Desktop binary confirmed at: `C:\Program Files\Docker\Docker\Docker Desktop.exe`
> Docker CLI confirmed at: `C:\Program Files\Docker\Docker\resources\bin\docker.exe`

After installation, **launch Docker Desktop** from the Start menu and:
1. Complete the first-run setup wizard
2. Accept the license agreement
3. Let it finish initializing (the whale icon in the taskbar turns solid when ready)
4. Verify in a new terminal: `docker --version`

### Step 3 — Install Python 3.11

```powershell
winget install --id Python.Python.3.11 --silent --accept-package-agreements --accept-source-agreements
```

> **What happened on this device:** Python 3.11.9 installed (25 MB download).
> Verified with: `python --version` → `Python 3.11.9`

Open a **new** PowerShell window so the PATH update takes effect.

### Step 4 — Install uv (Package Manager)

```powershell
winget install --id astral-sh.uv --silent --accept-package-agreements --accept-source-agreements
```

> **What happened on this device:** uv 0.11.13 installed (22.2 MB download).
> Also installed VC++ Redistributable as a dependency automatically.
> Verified with: `uv --version` → `uv 0.11.13`

Open a new PowerShell window after this too.

### ✅ Prerequisites Summary

```
WSL2          ✅  (active after reboot)
Docker Desktop ✅  C:\Program Files\Docker\Docker\Docker Desktop.exe
Python 3.11.9  ✅
uv 0.11.13     ✅
```

---

## 3. Project Structure Overview

```
FastAPI-boilerplate/
├── src/
│   ├── app/                   ← FastAPI application code
│   │   ├── api/v1/            ← Route handlers
│   │   ├── core/              ← Config, DB, security, worker
│   │   ├── crud/              ← Database operations (FastCRUD)
│   │   ├── models/            ← SQLAlchemy ORM models
│   │   └── schemas/           ← Pydantic request/response models
│   ├── migrations/            ← Alembic migration files
│   ├── scripts/               ← One-off scripts (superuser, tier)
│   ├── .env.local             ← ✅ Created by us (local config)
│   ├── .env.staging           ← ✅ Created by us (staging config)
│   └── .env.production        ← ✅ Created by us (production config)
│
├── scripts/                   ← Environment-specific Docker configs
│   ├── local_with_uvicorn/
│   │   ├── Dockerfile
│   │   ├── docker-compose.yml
│   │   └── .env.example
│   ├── gunicorn_managing_uvicorn_workers/
│   │   ├── Dockerfile
│   │   ├── docker-compose.yml
│   │   └── .env.example
│   └── production_with_nginx/
│       ├── Dockerfile
│       ├── docker-compose.yml
│       └── .env.example
│
├── default.conf               ← Nginx config (used by production)
├── uv.lock                    ← Locked dependency versions
├── pyproject.toml             ← Project metadata + dependencies
└── SETUP_GUIDE.md             ← This file
```

---

## 4. Environment Files Explained

We created three ready-to-use `.env` files in `src/`. Each environment has different:

| Setting | Local | Staging | Production |
|---------|-------|---------|------------|
| `ENVIRONMENT` | `local` | `staging` | `production` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | 60 | 30 | 15 |
| `CLIENT_CACHE_MAX_AGE` | 60s | 300s | 600s |
| `CORS_ORIGINS` | `["*"]` | staging domain | your domain |
| `POSTGRES_DB` | `fastapi_local` | `fastapi_staging` | `fastapi_production` |
| `SECRET_KEY` | unique generated | unique generated | unique generated |

> **Security:** All three `SECRET_KEY` values were generated with `secrets.token_hex(32)` (256-bit entropy). Never share or commit these values.

### Key env vars to know

```bash
POSTGRES_SERVER="db"       # Docker service name, not localhost!
REDIS_CACHE_HOST="redis"   # Docker service name, not localhost!
ENVIRONMENT="local"        # Controls app behavior (logging, docs visibility, etc.)
SECRET_KEY=<hex>           # Signs JWT tokens — must be unique per environment
```

---

## 5. Local Environment (Uvicorn + Hot-Reload)

**Best for:** Writing code. Changes to `src/app/` reload the server automatically.

### Activate the local config

```powershell
# From the project root:
Copy-Item src\.env.local src\.env
```

### Copy the local Docker files to project root

```powershell
Copy-Item scripts\local_with_uvicorn\Dockerfile Dockerfile -Force
Copy-Item scripts\local_with_uvicorn\docker-compose.yml docker-compose.yml -Force
```

### What the local Dockerfile does

Uses the modern **uv-based multi-stage build**:
1. **Builder stage** — `ghcr.io/astral-sh/uv:python3.11-bookworm-slim`: installs all dependencies via uv (cached layers)
2. **Final stage** — `python:3.11-slim-bookworm`: copies only the `.venv`, runs as non-root user `app`

### Build and start

```powershell
docker compose up --build
```

What spins up:
- `web` → Uvicorn on port `8000` with `--reload`
- `worker` → ARQ background job worker
- `db` → PostgreSQL 13 (data persisted in `postgres-data` volume)
- `redis` → Redis Alpine (data persisted in `redis-data` volume)

### Verify it's running

```powershell
# Health check (note the /api prefix — it's /api/v1/, NOT /v1/)
curl http://localhost:8000/api/v1/health

# Interactive API docs
start http://localhost:8000/docs
```

### Run tests (optional)

The `pytest` service is defined but commented out. To run tests:

```powershell
docker compose run --rm pytest
```

Or uncomment the `pytest` block in `docker-compose.yml` and run `docker compose up pytest`.

### Stop

```powershell
docker compose down
# To also wipe the database volume:
docker compose down -v
```

---

## 6. Staging Environment (Gunicorn + 4 Workers)

**Best for:** Testing production-like behavior without hot-reload. Multiple Uvicorn workers handle concurrent requests.

### Activate the staging config

```powershell
Copy-Item src\.env.staging src\.env
```

### Copy the staging Docker files

```powershell
Copy-Item scripts\gunicorn_managing_uvicorn_workers\Dockerfile Dockerfile -Force
Copy-Item scripts\gunicorn_managing_uvicorn_workers\docker-compose.yml docker-compose.yml -Force
```

### What's different from local

The `web` service command changes from:
```
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```
to:
```
gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8000
```

- `-w 4` → 4 worker processes (each can handle requests concurrently)
- No `--reload` → code changes require a container restart
- Still exposed on port `8000` directly (no Nginx in front)

> **Note:** All Dockerfiles (local, staging, production) now use the **uv-based multi-stage build** — the legacy Poetry-based Dockerfiles were broken (no `poetry.lock` in this repo) and have been rewritten.

### Build and start

```powershell
docker compose up --build
```

### Verify

```powershell
curl http://localhost:8000/api/v1/health
start http://localhost:8000/docs
```

### Stop

```powershell
docker compose down
```

---

## 7. Production Environment (Gunicorn + Nginx)

**Best for:** Real deployments. Nginx sits in front as a reverse proxy — it handles SSL termination, static files, and connection management. The `web` container is no longer exposed directly.

### Activate the production config

```powershell
Copy-Item src\.env.production src\.env
```

### Copy the production Docker files

```powershell
Copy-Item scripts\production_with_nginx\Dockerfile Dockerfile -Force
Copy-Item scripts\production_with_nginx\docker-compose.yml docker-compose.yml -Force
```

### Key differences from staging

**Ports:** `web` uses `expose: "8000"` instead of `ports: "8000:8000"`. This means port 8000 is only reachable inside the Docker network — external traffic must go through Nginx on port 80.

**Nginx service** is enabled:
```yaml
nginx:
  image: nginx:latest
  ports:
    - "80:80"
  volumes:
    - ./default.conf:/etc/nginx/conf.d/default.conf
  depends_on:
    - web
```

**Nginx config** (`default.conf` at project root):
```nginx
server {
    listen 80;
    location / {
        proxy_pass http://web:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Build and start

```powershell
docker compose up --build
```

Services that start:
- `nginx` → port 80 (public entry point)
- `web` → Gunicorn + 4 workers (internal, port 8000)
- `worker` → ARQ background jobs
- `db` → PostgreSQL
- `redis` → Redis

### Verify

```powershell
# Traffic goes through Nginx on port 80
curl http://localhost/api/v1/health
```

### Adding HTTPS (production-only note)

To add SSL with Let's Encrypt, update `default.conf`:

```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ...
}
server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

And mount the certs into the nginx container:
```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
  - ./default.conf:/etc/nginx/conf.d/default.conf
```

### Stop

```powershell
docker compose down
```

---

## 8. Post-Startup Steps (Every Environment)

These steps apply to whichever environment you just started. Run them **after** `docker compose up --build` succeeds.

### Step A — Run database migrations

Alembic migrations create all database tables.

> ⚠️ **Important:** The `web` container only mounts `./src/app` — it cannot see `alembic.ini` or `migrations/`. Running `docker compose exec web alembic ...` will fail with `No 'script_location' key found`. You must use the `create_superuser` container (which mounts all of `src/`) and `cd` to `/code/src` first.

**Local / Staging** (`create_superuser` service is available):
```powershell
docker compose run --rm create_superuser sh -c "cd /code/src && alembic upgrade head"
```

**Production** (`create_superuser` is commented out in the docker-compose — use `web` with an explicit volume):
```powershell
docker compose run --rm -v "${PWD}/src:/code/src" web sh -c "cd /code/src && alembic upgrade head"
```

> **Why `cd /code/src`?** Alembic resolves `script_location = migrations` relative to the current working directory, not the config file location.

### Step B — Create the first superuser

A superuser is required to access the admin panel and privileged endpoints.

**Local / Staging** (the `create_superuser` service has the right command baked in):
```powershell
docker compose run --rm create_superuser
```

**Production** (`create_superuser` is commented out — use `web` with volume):
```powershell
docker compose run --rm -v "${PWD}/src:/code/src" web python -m src.scripts.create_first_superuser
```

Credentials come from `.env`:
```
ADMIN_USERNAME=admin
ADMIN_PASSWORD=<your env password>
ADMIN_EMAIL=<your env email>
```

### Step C — Create the first tier

Tiers are subscription levels (free, pro, etc.) used for rate limiting and feature gating.

**Local / Staging:**
```powershell
docker compose run --rm create_superuser python -m src.scripts.create_first_tier
```

**Production:**
```powershell
docker compose run --rm -v "${PWD}/src:/code/src" web python -m src.scripts.create_first_tier
```

The tier name comes from `.env`: `TIER_NAME="free"`

### Step D — Verify everything

```powershell
# 1. Health check  (prefix is /api/v1/, NOT /v1/)
curl http://localhost:8000/api/v1/health    # local / staging
curl http://localhost/api/v1/health         # production (port 80)

# 2. Check all containers are healthy
docker compose ps

# 3. View logs
docker compose logs web
docker compose logs worker
docker compose logs db
```

---

## 9. Switching Between Environments

Since all three environments share the same `src/.env`, `Dockerfile`, and `docker-compose.yml` paths, switching is a 3-command process:

```powershell
# Stop current environment
docker compose down

# Switch to a different environment (example: local → staging)
Copy-Item src\.env.staging src\.env
Copy-Item scripts\gunicorn_managing_uvicorn_workers\Dockerfile Dockerfile -Force
Copy-Item scripts\gunicorn_managing_uvicorn_workers\docker-compose.yml docker-compose.yml -Force

# Start the new environment
docker compose up --build
```

### Environment quick-reference

| Environment | .env file | Dockerfile source | docker-compose.yml source |
|-------------|-----------|-------------------|--------------------------|
| Local | `src\.env.local` | `scripts\local_with_uvicorn\Dockerfile` | `scripts\local_with_uvicorn\docker-compose.yml` |
| Staging | `src\.env.staging` | `scripts\gunicorn_managing_uvicorn_workers\Dockerfile` | `scripts\gunicorn_managing_uvicorn_workers\docker-compose.yml` |
| Production | `src\.env.production` | `scripts\production_with_nginx\Dockerfile` | `scripts\production_with_nginx\docker-compose.yml` |

---

## 10. Troubleshooting

### Docker Desktop won't start
**Cause:** WSL2 not active (reboot required).  
**Fix:** Reboot and try again. Run `wsl --status` to confirm WSL2 is active.

### `docker: command not found` after install
**Cause:** PATH not refreshed.  
**Fix:** Open a **new** terminal window. Docker CLI is at:
`C:\Program Files\Docker\Docker\resources\bin\docker.exe`

### Port 8000 already in use
**Fix:**
```powershell
# Find what's using the port
netstat -ano | findstr :8000
# Kill it (replace PID)
taskkill /PID <PID> /F
```

### Database connection refused at startup
**Cause:** The `web` container starts before Postgres is ready.  
**Fix:** Add a `healthcheck` to the `db` service, or simply wait 10–15 seconds and run:
```powershell
docker compose restart web
```

### Alembic `Target database is not up to date` / `No script_location key`
```powershell
# Local / staging:
docker compose run --rm create_superuser sh -c "cd /code/src && alembic upgrade head"
# Production:
docker compose run --rm -v "${PWD}/src:/code/src" web sh -c "cd /code/src && alembic upgrade head"
```
> Do NOT use `docker compose exec web alembic ...` — the web container can't see `alembic.ini`.

### Changes to `src/app/` not reflecting (staging/production)
**Cause:** Gunicorn doesn't hot-reload.  
**Fix:** Restart the web container:
```powershell
docker compose restart web
```

### `SECRET_KEY` must be regenerated for production
Never use the keys in `.env.example`. To generate a fresh key:
```powershell
python -c "import secrets; print(secrets.token_hex(32))"
```

### View real-time logs
```powershell
docker compose logs -f web       # FastAPI app
docker compose logs -f worker    # ARQ worker
docker compose logs -f nginx     # Nginx (production only)
```

---

## Appendix — What Each Service Does

| Service | Image | Role |
|---------|-------|------|
| `web` | Built from `Dockerfile` | FastAPI app (Uvicorn or Gunicorn) |
| `worker` | Built from `Dockerfile` | ARQ async job queue consumer |
| `db` | `postgres:13` | Primary data store |
| `redis` | `redis:alpine` | Cache + job queue + rate-limit counters |
| `nginx` | `nginx:latest` | Reverse proxy (production only) |
| `create_superuser` | Built from `Dockerfile` | One-shot: seeds admin user |
| `create_tier` | Built from `Dockerfile` | One-shot: seeds first subscription tier |
| `pytest` | Built from `Dockerfile` | One-shot: runs the test suite |

---

## Appendix — API Endpoints

All routes are prefixed `/api/v1/` — two levels: `/api` from `api/__init__.py` and `/v1` from `api/v1/__init__.py`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/health` | Health check |
| POST | `/api/v1/login` | Get JWT tokens |
| POST | `/api/v1/logout` | Blacklist token |
| GET/POST | `/api/v1/users/` | User management |
| GET/POST | `/api/v1/posts/` | Posts (example CRUD model) |
| POST | `/api/v1/tasks` | Submit background task |
| GET/POST | `/api/v1/tiers/` | Subscription tier management |
| GET | `/api/v1/rate-limits/` | View rate limit rules |

Interactive docs at `/docs` (Swagger UI) — available in local and staging; disable for production by configuring the FastAPI app constructor.

---

*Guide written from real-time setup on this device — Windows 11, May 2026.*  
*Tools installed: WSL2, Docker Desktop, Python 3.11.9, uv 0.11.13*
