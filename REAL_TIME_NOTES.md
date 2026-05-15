# Real-Time Setup Notes
> Every command run, every error hit, every fix applied — in exact order.  
> Device: Windows 11 (Lenovo), fresh machine, no prior installs.  
> Date: 15 May 2026

---

## PHASE 0 — Pre-Reboot (before PC restart)

### 1. Enable WSL2

```powershell
wsl --install --no-distribution
```

**Output:**
```
Windows Subsystem for Linux has been installed.
The requested operation is successful. Changes will not be effective until the system is rebooted.
```

**Note:** Needed admin elevation. The `--no-distribution` flag installs the WSL2 kernel only — skips downloading Ubuntu/Debian, since Docker Desktop brings its own `docker-desktop` distro.

---

### 2. Install Docker Desktop

```powershell
winget install --id Docker.DockerDesktop --silent --accept-package-agreements --accept-source-agreements
```

**Output:** Successfully installed (ran in background during Python install).  
**Install location:** `C:\Program Files\Docker\Docker\Docker Desktop.exe`  
**CLI location:** `C:\Program Files\Docker\Docker\resources\bin\docker.exe`

---

### 3. Install Python 3.11

```powershell
winget install --id Python.Python.3.11 --silent --accept-package-agreements --accept-source-agreements
```

**Output:** Python 3.11.9 installed (25.0 MB download from python.org).  
**Verified with:** `python --version` → `Python 3.11.9`

---

### 4. Install uv (package manager)

```powershell
winget install --id astral-sh.uv --silent --accept-package-agreements --accept-source-agreements
```

**Output:** uv 0.11.13 installed (22.2 MB download). Also installed VC++ Redistributable as a dependency automatically.  
**Verified with:** `uv --version` → `uv 0.11.13 (4512a3931 2026-05-10 x86_64-pc-windows-msvc)`

---

### 5. Generate secure secret keys (Python)

```powershell
python -c "import secrets; print(secrets.token_hex(32))"
# Run 3 times — one per environment
```

| Environment | Secret Key |
|-------------|-----------|
| Local | `fff4950deb8444faef0b701de9eff9297696232ac4ed878a6f605b2a7d811e9e` |
| Staging | `29695c1d6ac6654fed7ed14bafa8160177968797817be0fc2b3116d9b8f9f52f` |
| Production | `fa595ea6b423df15992a0917b59c6ea4190f843dcca46698ea2bf8035adf21d9` |

---

### 6. Create environment files

Created three files in `src/`:

| File | Purpose |
|------|---------|
| `src/.env.local` | Local dev — Uvicorn, open CORS, 60min token, DB: `fastapi_local` |
| `src/.env.staging` | Staging — Gunicorn 4w, restricted CORS, 30min token, DB: `fastapi_staging` |
| `src/.env.production` | Production — Gunicorn 4w + Nginx, strict CORS, 15min token, DB: `fastapi_production` |

**Key differences across envs:**
```
ENVIRONMENT=local|staging|production
ACCESS_TOKEN_EXPIRE_MINUTES=60|30|15
CLIENT_CACHE_MAX_AGE=60|300|600
CORS_ORIGINS=["*"] | [staging-domain] | [prod-domain]
POSTGRES_DB=fastapi_local|fastapi_staging|fastapi_production
SECRET_KEY=<unique per env>
```

---

### ⚠️ REBOOT REQUIRED HERE

WSL2 kernel is installed but inactive until reboot.  
**Action:** Restarted the PC.

---

## PHASE 1 — Post-Reboot

### 7. Verify everything is active after reboot

```powershell
wsl --status
# Output: Default Distribution: docker-desktop | Default Version: 2

wsl --list --verbose
# Output: docker-desktop   Running   2  ✅

docker version --format "Client: {{.Client.Version}} / Server: {{.Server.Version}}"
# Output: Client: 29.4.3 / Server: 29.4.3  ✅

python --version   # Python 3.11.9  ✅
uv --version       # uv 0.11.13     ✅
```

**Note:** Docker Desktop must be launched from Start menu first — the whale icon in the taskbar must be solid (not animated) before `docker` commands work.

---

## PHASE 2 — Local Environment Setup

### 8. Activate local config files

```powershell
# Copy the right .env for local
Copy-Item src\.env.local src\.env -Force

# Copy local-specific Dockerfile (uv-based, Uvicorn + hot-reload)
Copy-Item scripts\local_with_uvicorn\Dockerfile Dockerfile -Force

# Copy local docker-compose (web on port 8000, hot-reload)
Copy-Item scripts\local_with_uvicorn\docker-compose.yml docker-compose.yml -Force
```

**Why these files?**
- `scripts/local_with_uvicorn/Dockerfile` uses the modern uv multi-stage build
- Server command: `uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload`
- Port 8000 exposed directly (no reverse proxy)

---

### 9. Build local images

```powershell
docker compose build --no-cache
```

**What happened:**
- Downloaded base images: `ghcr.io/astral-sh/uv:python3.11-bookworm-slim` + `python:3.11-slim-bookworm`
- uv installed 74 packages into `.venv` (psycopg2, sqlalchemy, fastapi, redis, arq, etc.)
- Built 4 images: `web`, `worker`, `create_superuser`, `pytest`
- Total build time: ~35 seconds

**Images built:**
```
fastapi-boilerplate-web           ✅
fastapi-boilerplate-worker        ✅
fastapi-boilerplate-create_superuser ✅
fastapi-boilerplate-pytest        ✅
```

---

### 10. Start local containers (db + redis first, then app)

```powershell
# Start infrastructure first — give DB time to initialise
docker compose up -d db redis

# Wait for DB to accept connections
Start-Sleep -Seconds 8

# Now start the app
docker compose up -d web worker
```

**Why staged start?** PostgreSQL takes ~5 seconds to initialise. If the app starts too early it gets `Connection refused` on startup. The `depends_on` in docker-compose handles ordering but not readiness — a manual wait or `healthcheck` is cleaner.

**Container status:**
```
fastapi-boilerplate-db-1      postgres:13       Up   5432/tcp (internal)
fastapi-boilerplate-redis-1   redis:alpine      Up   6379/tcp (internal)
fastapi-boilerplate-web-1     ...web            Up   0.0.0.0:8000->8000/tcp
fastapi-boilerplate-worker-1  ...worker         Up   (no port — ARQ internal)
```

---

### 11. Run Alembic migrations

**First attempt (FAILED):**
```powershell
docker compose exec web alembic -c src/alembic.ini upgrade head
```
**Error:** `FAILED: No 'script_location' key found in configuration.`  
**Cause:** The `web` container only mounts `./src/app:/code/app`. The `alembic.ini` and `migrations/` directory are NOT inside `app/` — they sit in `src/`. The web container can't see them.

**Fix — use `create_superuser` which mounts all of `src/`:**
```powershell
# First verify the container can see alembic files
docker compose run --rm create_superuser ls /code/src
# Output: __init__.py  alembic.ini  app  migrations  scripts ✅

# Run from /code/src where alembic.ini lives
docker compose run --rm create_superuser sh -c "cd /code/src && alembic upgrade head"
```

**Output:**
```
INFO  [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO  [alembic.runtime.migration] Will assume transactional DDL.
# Exit code: 0 ✅
```

**Why `cd /code/src`?** Alembic resolves `script_location = migrations` relative to the current working directory, not the config file location. Running from `/code/src` means `migrations/` resolves to `/code/src/migrations/`.

---

### 12. Create first superuser

```powershell
docker compose run --rm create_superuser python -m src.scripts.create_first_superuser
```

**Output:** `INFO:__main__:Admin user admin created successfully.`  
**Credentials come from `.env`:** `ADMIN_USERNAME=admin`, `ADMIN_PASSWORD=Local@Admin123!`

---

### 13. Create first tier (with bug fix)

**First attempt (FAILED):**
```powershell
docker compose run --rm create_superuser python -m src.scripts.create_first_tier
```
**Error:** `ImportError: cannot import name 'config' from 'src.app.core.config'`  
**Cause:** `create_first_tier.py` had `from ..app.core.config import config` — but the module only exports `settings = Settings()`, not `config`.

**Second attempt after import fix (FAILED):**
Changed to `from ..app.core.config import settings` and `tier_name = settings.TIER_NAME`.  
**Error:** `'Settings' object has no attribute 'TIER_NAME'`  
**Cause:** `TIER_NAME` is in `.env` but not defined in any `Settings` class. Pydantic ignores unknown env vars when `extra="ignore"`.

**Final fix — use `os.getenv`:**
```python
# src/scripts/create_first_tier.py
import os
# ...
tier_name = os.getenv("TIER_NAME", "free")
```

```powershell
docker compose run --rm create_superuser python -m src.scripts.create_first_tier
```
**Output:** `INFO:__main__:Tier 'free' created successfully.` ✅

---

### 14. Verify local environment

**First attempt (FAILED):**
```powershell
Invoke-WebRequest -Uri "http://localhost:8000/v1/health"
# Error: 404 Not Found
```
**Cause:** Wrong URL. Checked the `api/__init__.py` — the router uses prefix `/api`, and v1 router uses `/v1`. Full path is `/api/v1/health`, not `/v1/health`.

**Correct URL:**
```powershell
Invoke-WebRequest -Uri "http://localhost:8000/api/v1/health"
# Status: 200
# Body: {"status":"healthy","environment":"local","version":"0.1","timestamp":"..."}
```
✅ **LOCAL ENVIRONMENT CONFIRMED WORKING**

---

### 15. Tear down local

```powershell
docker compose down -v
# -v removes named volumes (postgres-data, redis-data)
# Fresh DB for each environment — avoids data contamination
```

---

## PHASE 3 — Staging Environment Setup

### 16. Activate staging config files

```powershell
Copy-Item src\.env.staging src\.env -Force
Copy-Item scripts\gunicorn_managing_uvicorn_workers\Dockerfile Dockerfile -Force
Copy-Item scripts\gunicorn_managing_uvicorn_workers\docker-compose.yml docker-compose.yml -Force
```

---

### 17. Build staging images (FAILED — Dockerfile used Poetry)

```powershell
docker compose build --no-cache
```

**Error:**
```
RUN poetry export -f requirements.txt --output requirements.txt --without-hashes
ERROR: process did not complete successfully: exit code: 1
```

**Cause:** The `scripts/gunicorn_managing_uvicorn_workers/Dockerfile` is a legacy Poetry-based Dockerfile. The project has migrated to `uv` (`uv.lock` exists, no `poetry.lock`). `poetry export` fails without a lock file.

**Fix — rewrote Dockerfile to use uv, kept Gunicorn CMD:**
```dockerfile
# Same uv multi-stage build as local, but CMD is gunicorn:
FROM ghcr.io/astral-sh/uv:python3.11-bookworm-slim AS builder
# ... (same uv sync steps) ...
FROM python:3.11-slim-bookworm
# ...
CMD ["gunicorn", "app.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

**Rebuild (fast — uv cache hit):**
```powershell
docker compose build
# Time: ~10 seconds (cached layers from local build)
```
✅ All 3 images built.

---

### 18. Start staging containers + seed

```powershell
docker compose up -d db redis
Start-Sleep -Seconds 8
docker compose up -d web worker

# Verify Gunicorn is the server (not Uvicorn)
docker compose ps
# web COMMAND: "gunicorn app.main:a…" ✅

# Migrations
docker compose run --rm create_superuser sh -c "cd /code/src && alembic upgrade head"

# Superuser
docker compose run --rm create_superuser python -m src.scripts.create_first_superuser

# Tier
docker compose run --rm create_superuser python -m src.scripts.create_first_tier
```

**All outputs:** success ✅

---

### 19. Verify staging environment

```powershell
Invoke-WebRequest -Uri "http://localhost:8000/api/v1/health"
# Status: 200
# Body: {"status":"healthy","environment":"staging","version":"0.1","timestamp":"..."}
```
✅ **STAGING ENVIRONMENT CONFIRMED WORKING**

---

### 20. Tear down staging

```powershell
docker compose down -v
```

---

## PHASE 4 — Production Environment Setup

### 21. Activate production config files

```powershell
Copy-Item src\.env.production src\.env -Force
Copy-Item scripts\production_with_nginx\docker-compose.yml docker-compose.yml -Force
# Dockerfile stays the same (already fixed to uv + gunicorn)
```

**Key differences in production docker-compose vs staging:**
- `web` uses `expose: "8000"` instead of `ports: "8000:8000"` → port 8000 is INTERNAL ONLY
- `nginx` service added: `ports: "80:80"`, proxies to `http://web:8000`
- `create_superuser` service is commented out

---

### 22. Build production images

```powershell
docker compose build
# Time: ~instant (fully cached — same Dockerfile as staging)
```

---

### 23. Start production containers

```powershell
docker compose up -d db redis
Start-Sleep -Seconds 8
docker compose up -d web worker nginx

docker compose ps
# nginx-1  nginx:latest  0.0.0.0:80->80/tcp  ✅ (public entry point)
# web-1    ...web        8000/tcp             ✅ (internal only, no public mapping)
```

---

### 24. Seed production database

**Problem:** `create_superuser` service is commented out in production docker-compose.

```powershell
docker compose run --rm create_superuser sh -c "..."
# Error: no such service: create_superuser
```

**Fix — use `web` service with extra volume mount:**
```powershell
# Migrations
docker compose run --rm -v "${PWD}/src:/code/src" web sh -c "cd /code/src && alembic upgrade head"

# Superuser
docker compose run --rm -v "${PWD}/src:/code/src" web python -m src.scripts.create_first_superuser

# Tier
docker compose run --rm -v "${PWD}/src:/code/src" web python -m src.scripts.create_first_tier
```

All three: ✅

---

### 25. Production: Gunicorn race condition on first boot

**Problem:** Gunicorn spawns 4 workers simultaneously. Each worker runs the FastAPI lifespan which calls CRUDAdmin's `create_initial_admin_user`. All 4 hit the DB at the same time → unique constraint violation on `admin_user` → one worker crashes with `IntegrityError` → Gunicorn kills the rest.

**Error in logs:**
```
[ERROR] Worker (pid:7) exited with code 3
[ERROR] Worker failed to boot.
# Result: 502 Bad Gateway from Nginx
```

**Fix — restart web after first boot (admin user now exists, no more conflicts):**
```powershell
docker compose restart web
Start-Sleep -Seconds 5
```

**On subsequent boots:** CRUDAdmin finds the existing admin user and skips creation → no conflict → all 4 workers boot cleanly.

---

### 26. Verify production environment

```powershell
# Traffic goes THROUGH Nginx on port 80 (not directly to port 8000)
Invoke-WebRequest -Uri "http://localhost/api/v1/health"
# Status: 200
# Body: {"status":"healthy","environment":"production","version":"0.1","timestamp":"..."}

# Confirm port 8000 is NOT accessible directly
Invoke-WebRequest -Uri "http://localhost:8000/api/v1/health" -TimeoutSec 3
# Error: Connection refused ✅ (correct — internal only)
```

✅ **PRODUCTION ENVIRONMENT CONFIRMED WORKING**

---

## PHASE 5 — Repo Cleanup (OCD Pass)

After all three environments were verified working, a full cleanup pass was run to remove every broken, redundant, or insecure file.

### 27. Delete build artifact

```powershell
Remove-Item "C:\Projects\FastAPI-boilerplate\build_local.log"
```

`build_local.log` was created by redirecting Docker build output during staging setup. It has no place in the repo.

---

### 28. Delete runtime `src/.env`

```powershell
Remove-Item "C:\Projects\FastAPI-boilerplate\src\.env"
```

Already listed in `.gitignore` — it's a runtime-generated file that containers read at startup. It was left on disk from the last environment's run.

---

### 29. Fix `.gitignore` — add secret env files

**Problem:** `src/.env.local`, `src/.env.staging`, `src/.env.production` contain real `SECRET_KEY` values but were not in `.gitignore`.

```gitignore
# Config files (never commit secrets):
src/.env
src/.env.local
src/.env.staging
src/.env.production
```

---

### 30. Rewrite `scripts/gunicorn_managing_uvicorn_workers/Dockerfile`

This is the *source* Dockerfile for staging. It was still Poetry-based (broken). Rewrote it to use the same uv multi-stage build with Gunicorn CMD.

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.11-bookworm-slim AS builder
ENV UV_COMPILE_BYTECODE=1
ENV UV_LINK_MODE=copy
WORKDIR /app
RUN --mount=type=cache,target=/root/.cache/uv \
    --mount=type=bind,source=uv.lock,target=uv.lock \
    --mount=type=bind,source=pyproject.toml,target=pyproject.toml \
    uv sync --locked --no-install-project
COPY . /app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-editable
FROM python:3.11-slim-bookworm
RUN groupadd --gid 1000 app && useradd --uid 1000 --gid app --shell /bin/bash --create-home app
COPY --from=builder --chown=app:app /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
USER app
WORKDIR /code
CMD ["gunicorn", "app.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

---

### 31. Rewrite `scripts/production_with_nginx/Dockerfile`

Two bugs in one file:
1. Poetry-based (broken, no `poetry.lock`)
2. Active CMD was `uvicorn ... --reload` — hot-reload in production!

Same uv multi-stage rewrite, same Gunicorn CMD as above.

---

### 32. Run ruff linter

```powershell
# Install ruff
uv tool install ruff

# Auto-fix safe issues
& "C:\Users\Lenovo\.local\bin\ruff.exe" check "C:\Projects\FastAPI-boilerplate\src" --fix --statistics
# Output: Found 9 errors (7 fixed, 2 remaining).
# 2 remaining = UP042 (replace-str-enum) — require --unsafe-fixes

# Find the 2 remaining files
& "C:\Users\Lenovo\.local\bin\ruff.exe" check "C:\Projects\FastAPI-boilerplate\src" --select UP042
# src\app\core\config.py:162   EnvironmentOption(str, Enum)
# src\app\core\security.py:24  TokenType(str, Enum)
```

**Manually fixed both files:**

`src/app/core/config.py`:
```python
# Before:
from enum import Enum
class EnvironmentOption(str, Enum):

# After:
from enum import StrEnum
class EnvironmentOption(StrEnum):
```

`src/app/core/security.py`:
```python
# Before:
from enum import Enum
class TokenType(str, Enum):

# After:
from enum import StrEnum
class TokenType(StrEnum):
```

**Final ruff check:**
```powershell
& "C:\Users\Lenovo\.local\bin\ruff.exe" check "C:\Projects\FastAPI-boilerplate\src"
# All checks passed!
# Exit code: 0  ✅
```

---

### Phase 5 Summary

| Action | File | Reason |
|--------|------|--------|
| Deleted | `build_local.log` | Docker build artifact, not repo content |
| Deleted | `src/.env` | Gitignored runtime file left on disk |
| Fixed | `.gitignore` | Added secret env files that were missing |
| Rewritten | `scripts/gunicorn_managing_uvicorn_workers/Dockerfile` | Legacy Poetry → uv |
| Rewritten | `scripts/production_with_nginx/Dockerfile` | Poetry + wrong `--reload` CMD → uv + Gunicorn |
| Fixed | `src/app/core/config.py` | `(str, Enum)` → `StrEnum` (ruff UP042) |
| Fixed | `src/app/core/security.py` | `(str, Enum)` → `StrEnum` (ruff UP042) |
| Auto-fixed | 7 other files | Various ruff violations (safe auto-fix) |

---

## SUMMARY — All Environments

| Env | Server | Port | Health URL | Status |
|-----|--------|------|------------|--------|
| Local | Uvicorn + hot-reload | 8000 | `http://localhost:8000/api/v1/health` | ✅ |
| Staging | Gunicorn 4 workers | 8000 | `http://localhost:8000/api/v1/health` | ✅ |
| Production | Gunicorn 4w + Nginx | **80** | `http://localhost/api/v1/health` | ✅ |

---

## BUGS FOUND AND FIXED

### Bug 1 — `create_first_tier.py` ImportError
**File:** `src/scripts/create_first_tier.py`  
**Original:** `from ..app.core.config import config`  
**Problem:** Module exports `settings`, not `config`  
**Fix:**
```python
# Before:
from ..app.core.config import config
tier_name = config("TIER_NAME", default="free")

# After:
import os
tier_name = os.getenv("TIER_NAME", "free")
```

### Bug 2 — Staging Dockerfile uses Poetry (no poetry.lock)
**File:** `scripts/gunicorn_managing_uvicorn_workers/Dockerfile` (copied to root `Dockerfile` during staging setup)  
**Problem:** Project migrated to uv but this Dockerfile still calls `poetry export`. Since there is no `poetry.lock`, the build fails immediately.  
**Fix:** Rewrote the root `Dockerfile` to use the uv multi-stage build — same builder stage as local, but CMD is Gunicorn instead of Uvicorn. (The `scripts/` source Dockerfiles were corrected during the cleanup phase — see Phase 5.)

### Bug 3 — Production `create_superuser` service commented out
**File:** `scripts/production_with_nginx/docker-compose.yml`  
**Problem:** Can't run `docker compose run --rm create_superuser ...` in production  
**Fix:** Use `web` service with extra volume mount: `docker compose run --rm -v "${PWD}/src:/code/src" web ...`

### Bug 4 — Gunicorn race condition on first boot (production)
**Root cause:** All 4 Gunicorn workers simultaneously try to create the CRUDAdmin `admin_user` → unique constraint error → worker crash  
**Fix:** `docker compose restart web` after first boot. Subsequent starts are clean.

### Bug 5 — `.gitignore` not protecting secret env files
**Files:** `src/.env.local`, `src/.env.staging`, `src/.env.production`  
**Problem:** All three files contain real 256-bit `SECRET_KEY` values generated for this machine — but `.gitignore` had no entries for them. They would have been committed on the next `git add .`.  
**Fix:** Added all three paths to `.gitignore` under the `# Config files (never commit secrets)` section.

### Bug 6 — Production `scripts/` Dockerfile had wrong server CMD
**File:** `scripts/production_with_nginx/Dockerfile`  
**Problem:** The CMD was `uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload` — hot-reload enabled in a production container. This is both incorrect (Gunicorn should manage workers) and a security/performance risk.  
**Fix:** Rewrote the Dockerfile to use uv multi-stage build + correct Gunicorn CMD:
```dockerfile
CMD ["gunicorn", "app.main:app", "-w", "4", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```

### Bug 7 — Ruff linter violations in source code
**Files:** `src/app/core/config.py`, `src/app/core/security.py` (UP042), plus 7 other auto-fixable issues  
**Problem:** Both files used `class Foo(str, Enum)` — the legacy pattern. Python 3.11+ has `StrEnum` which is cleaner and eliminates a redundant base class.  
**Fix:**
```python
# Before:
from enum import Enum
class EnvironmentOption(str, Enum): ...
class TokenType(str, Enum): ...

# After:
from enum import StrEnum
class EnvironmentOption(StrEnum): ...
class TokenType(StrEnum): ...
```
Ruff final result: **`All checks passed!`** (exit code 0)

---

## ALEMBIC MIGRATION PATTERN

The `web` container only mounts `src/app/` — not `alembic.ini` or `migrations/`.

**Always run migrations via `create_superuser` (or with explicit volume mount):**

```powershell
# Local / Staging (create_superuser service available):
docker compose run --rm create_superuser sh -c "cd /code/src && alembic upgrade head"

# Production (create_superuser commented out):
docker compose run --rm -v "${PWD}/src:/code/src" web sh -c "cd /code/src && alembic upgrade head"
```

**Why `cd /code/src`?** Alembic resolves `script_location = migrations` relative to the CWD, not the config file directory.

---

## CORRECT API ENDPOINT PREFIX

All routes are prefixed with `/api/v1/` (not `/v1/`):
- `api/__init__.py` → `APIRouter(prefix="/api")`
- `api/v1/__init__.py` → `APIRouter(prefix="/v1")`

| Correct | Wrong |
|---------|-------|
| `http://localhost:8000/api/v1/health` | `http://localhost:8000/v1/health` |
| `http://localhost:8000/api/v1/login` | `http://localhost:8000/v1/login` |
| `http://localhost/api/v1/health` (prod) | `http://localhost/v1/health` |

Interactive docs: `http://localhost:8000/docs` (local/staging) or `http://localhost/docs` (prod)

---

## SWITCHING ENVIRONMENTS (Quick Reference)

```powershell
# 1. Stop current environment
docker compose down -v

# 2. Choose target environment and copy its files:
# --- LOCAL ---
Copy-Item src\.env.local src\.env -Force
Copy-Item scripts\local_with_uvicorn\Dockerfile Dockerfile -Force
Copy-Item scripts\local_with_uvicorn\docker-compose.yml docker-compose.yml -Force

# --- STAGING ---
Copy-Item src\.env.staging src\.env -Force
Copy-Item scripts\gunicorn_managing_uvicorn_workers\docker-compose.yml docker-compose.yml -Force
# (Dockerfile stays — already fixed to uv+gunicorn)

# --- PRODUCTION ---
Copy-Item src\.env.production src\.env -Force
Copy-Item scripts\production_with_nginx\docker-compose.yml docker-compose.yml -Force

# 3. Build and start
docker compose build
docker compose up -d db redis
Start-Sleep -Seconds 8
docker compose up -d web worker        # local/staging
docker compose up -d web worker nginx  # production only

# 4. Seed (local/staging use create_superuser service; production uses web with volume)
docker compose run --rm create_superuser sh -c "cd /code/src && alembic upgrade head"
docker compose run --rm create_superuser python -m src.scripts.create_first_superuser
docker compose run --rm create_superuser python -m src.scripts.create_first_tier

# 5. Production only — restart web after first boot (race condition fix)
docker compose restart web
```

---

*Notes captured in real time during actual setup — Windows 11, May 15 2026*
