# Deployment

> Cloud Run deployment model, migration safety path, Docker runtime, and CI/CD.

## Overview

Life OS core (inside Agent-C) runs on **Google Cloud Run** with PostgreSQL as primary state and optional Neo4j graph integration. Deployments use a single backend image and a pre-deploy migration stage to avoid startup race conditions.

## Production URLs

| Service | URL |
|---------|-----|
| Backend API | `https://ces-backend-331630083873.us-central1.run.app` |
| Landing Page | `https://agent-c.agency` |
| Web App | `https://app.agent-c.agency` (legacy: `https://app.writersroom.online`) |

## Build and Runtime Artifacts

### Backend image

**File:** `backend/Dockerfile`

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# System deps + Node.js for CLI agents
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl git ca-certificates gnupg \
    && # ... Node.js 22.x installation ...
    && npm install -g \
       @anthropic-ai/claude-code \
       @google/gemini-cli \
       @openai/codex \
       @qwen-code/qwen-code

COPY pyproject.toml .
RUN pip install --no-cache-dir .

COPY app/ app/
COPY config/ config/
COPY migrations/ migrations/
COPY alembic.ini .

ENV PORT=8080
EXPOSE ${PORT}

COPY entrypoint.sh .
RUN chmod +x entrypoint.sh
CMD ["./entrypoint.sh"]
```

The image includes runtime dependencies plus CLI tooling used by Director/agent workflows.

### Additional images

| File | Purpose |
|------|---------|
| `backend/Dockerfile.dev` | Development image with dev dependencies |
| `backend/Dockerfile.architect` | Standalone Architect agent |
| `backend/Dockerfile.audio` | PyTorch + Demucs for Vertex AI audio jobs |

## Deployment Pipeline (Cloud Build)

**File:** `cloudbuild.yaml` (repo root)

Current order:

1. Build backend image
2. Push image tags (`$COMMIT_SHA`, `latest`)
3. Run migrations from the built image via `app.core.migration_runner.run_migrations(...)`
4. Deploy Cloud Run revision with `CES_SKIP_AUTO_MIGRATE=true`

Important:

- `_DATABASE_URL` must be configured in Cloud Build trigger substitutions.
- Migration execution belongs to deploy-time; container startup migration is fallback-only.

## Entrypoint and Startup

**File:** `backend/entrypoint.sh`

The entrypoint handles:

### Director Mode setup (non-blocking)

If `CES_DIRECTOR_MODE=true` and `CES_GITHUB_PAT` is set, performs a sparse git clone:

1. Creates a temporary `GIT_ASKPASS` script (avoids embedding PAT in URLs)
2. Shallow clone with `--filter=blob:none --sparse`
3. Sparse checkout of key directories (backend, frontend, config, docs)
4. Fetches individual files outside cone mode (CLAUDE.md, WORK_INDEX.md)
5. Removes PAT from remote URL after clone
6. Cleans up askpass script

This entire block runs under `set +e` so failures are non-blocking -- the app starts regardless.

### Fallback migration path (safety only)

When `CES_SKIP_AUTO_MIGRATE` is not `true` and `DATABASE_URL` is available, entrypoint executes a timeout-capped fallback migration call through `migration_runner`. Failure is logged as warning and does not block process startup.

### Application start

```bash
exec uvicorn app.main:app \
    --host 0.0.0.0 \
    --port ${PORT:-8080} \
    --workers 2 \
    --timeout-keep-alive 30 \
    --access-log
```

## CI/CD

**File:** `.github/workflows/ci.yml`

### Jobs

| Job | Trigger | Steps |
|-----|---------|-------|
| `lint` | push/PR to main | ruff check, ruff format, unauthorized LLM import check |
| `test` | push/PR to main | pytest with aiosqlite backend |
| `notify-failure` | lint or test fails | Webhook to CI Fix Pipeline |

### LLM Import Guard

The CI pipeline includes a custom check that prevents services from importing `LLMService` or `LLMClient` directly. All tenant-scoped services must route through `BudgetOrchestrator`:

```bash
# Flags: from app.services.llm_service import
# Excludes: infrastructure files, non-tenant-scoped services
# Allows: # budget-gate-fallback comment
```

### CI Fix Pipeline

On failure, the `notify-failure` job sends an HMAC-signed webhook payload that triggers a two-AI fix pipeline. The payload includes run ID, branch, SHA, and conclusion.

## Landing Page Deployment

**File:** `.github/workflows/deploy-landing.yml`

Separate workflow for the marketing/landing page site. Deploys to the `agent-c.agency` domain.

## Critical Environment Variables

See [configuration.md](configuration.md) for the full list. Key deployment variables:

| Variable | Required | Purpose |
|----------|----------|---------|
| `CES_DATABASE_URL` | Yes | PostgreSQL connection string |
| `CES_NEO4J_URI` | No | Neo4j connection (app starts without it) |
| `CES_FIREBASE_PROJECT_ID` | Yes | Firebase auth (set to project ID) |
| `CES_ALLOW_DEV_AUTH` | No | Dev auth bypass (must be false in prod) |
| `CES_DIRECTOR_MODE` | No | Enable git operations |
| `CES_GITHUB_PAT` | No | GitHub token for Director Mode |
| `CES_SKIP_AUTO_MIGRATE` | Yes (prod) | Prevent startup migration path |
| `PORT` | Auto | Set by Cloud Run (default 8080) |

## Graceful Startup

**File:** `backend/app/main.py`

The application handles missing services gracefully:

- **Neo4j**: Wrapped in try/except. App starts without it, graph operations return empty results.
- **Scheduler**: Only starts if `CES_SCHEDULER_ENABLED=true`.
- **Telegram**: Only initializes if `CES_TELEGRAM_BOT_TOKEN` is set.

## Known Operational Notes

1. **Startup migration is fallback-only**: production should rely on Cloud Build migration stage.
2. **Neo4j remains optional**: app degrades gracefully when unavailable.
3. **Audio-heavy jobs**: routed to dedicated worker/image rather than general backend runtime.

## Verification Checklist

```bash
cd /Users/gch2024/Dev/context-engine-studio/backend
bash -n entrypoint.sh
python -m py_compile app/core/migration_runner.py app/main.py
cd /Users/gch2024/Dev/context-engine-studio
python -c "import yaml; yaml.safe_load(open('cloudbuild.yaml'))"
```

## Related Documents

- [configuration.md](configuration.md) -- All environment variables
- [auth-system.md](auth-system.md) -- Authentication setup
- [scheduler.md](scheduler.md) -- Cron job infrastructure
