# Architecture Overview

> Context Engine Studio (CES) is a cloud-hosted creative engine that helps creators produce stories, videos, comics, and newsletters using AI. It is a multi-tenant FastAPI backend backed by PostgreSQL and Neo4j, with a React frontend. All AI model access is abstracted behind a provider-agnostic interface supporting Gemini, Claude, and OpenAI via BYOK (Bring Your Own Key).

## Current-State Addendum (Feb 2026)

Recent architecture changes now in code:

- Workbench-first entry path with auth-gated login and `/workbench` landing.
- Agentic brain upgrade:
  - `AgenticLoop` now supports multi-tool sequential turns, retries, compaction, and configurable turn caps.
  - `ModelFallbackService` handles provider/model failover with backoff.
  - `AgentStreamBus` + Telegram progressive edits reduce perceived latency.
- Memory and retrieval upgrades:
  - `ConversationCrystallizerSkill` and `KnowledgeExtractor` convert imported conversation data into Codex candidates.
  - `RetrievalScorer` adds recency + importance + relevance ranking and query-aware pull path.
- New intelligence surfaces:
  - URL Intelligence (`/url-intelligence/*`) with run/analyze/memo/triage paths.
  - Inbox Router skill for MIME-based file classification and routing.
- Cloud deployment hardening:
  - migration execution moved out of startup path into controlled deployment flow with fallback behavior.

Updated flowcharts are maintained in:

- `docs/flows/2026-02-21-agent-c-life-os-flowcharts.md`

## System Architecture

```
                         ┌──────────────────────────┐
                         │       FRONTENDS           │
                         │  React SPA  │  Telegram   │
                         └──────┬──────┴──────┬──────┘
                                │             │
                         ┌──────▼─────────────▼──────┐
                         │     FastAPI (ASGI)         │
                         │  Middleware Stack          │
                         │  ┌─────────────────────┐  │
                         │  │ CORSMiddleware       │  │
                         │  │ RequestIDMiddleware   │  │
                         │  │ TenantContextMiddleware│  │
                         │  └─────────────────────┘  │
                         │                           │
                         │     86 API Routers        │
                         │     /api/v1/*             │
                         └──────┬──────┬──────┬──────┘
                                │      │      │
              ┌─────────────────┤      │      ├─────────────────┐
              ▼                 ▼      ▼      ▼                 ▼
     ┌────────────┐   ┌────────────┐ ┌───────────┐   ┌──────────────┐
     │  Service   │   │   Graph    │ │  Outbox   │   │   Sinks      │
     │  Layer     │   │   Layer    │ │  Relay    │   │   (18)       │
     │  (146+)    │   │  (Neo4j)   │ │           │   │              │
     └─────┬──────┘   └─────┬─────┘ └─────┬─────┘   └──────┬───────┘
           │                │             │                  │
           ▼                ▼             ▼                  ▼
     ┌──────────┐    ┌──────────┐   ┌──────────┐    ┌──────────────┐
     │PostgreSQL│    │  Neo4j 5 │   │  Event   │    │  External    │
     │ (pgvector)│   │  (APOC)  │   │  Store   │    │  APIs/Sinks  │
     └──────────┘    └──────────┘   └──────────┘    └──────────────┘
```

## Backend Structure

```
backend/
├── app/
│   ├── main.py              # FastAPI app, lifespan, middleware registration
│   ├── api/v1/              # 86 FastAPI routers (one per domain)
│   │   └── router.py        # Central router registry
│   ├── core/                # Config, database, security, auth, vault
│   ├── models/              # 60 SQLAlchemy models
│   ├── schemas/             # 30+ Pydantic request/response schemas
│   ├── services/            # 146+ business logic services
│   ├── sinks/               # 18 output destinations (Telegram, Medium, KDP, etc.)
│   ├── graph/               # Neo4j client, schema, tenant isolation
│   ├── middleware/          # TenantContext, TierGate
│   ├── events/              # Event sourcing models
│   ├── data/                # YAML configs (formats, frameworks)
│   └── migrations/          # 50+ Alembic migrations
├── tests/                   # Large pytest suite (unit, integration, API)
├── config/                  # Framework YAMLs, soul documents
├── Dockerfile               # Production container
├── Dockerfile.audio         # Demucs/PyTorch worker (Vertex AI)
├── Dockerfile.architect     # ClosedClaw agent container
└── entrypoint.sh            # Cloud Run startup (fallback migration + app start)
```

## Frontend Structure

```
frontend/src/
├── pages/               # 18 route-level components
├── components/          # 186 React components (grouped by domain)
│   ├── workbench/       # Makers' Workbench (central workspace)
│   ├── editor/          # Scene editor
│   ├── video/           # Video pipeline UI
│   ├── comic/           # Comics Mode
│   ├── onboarding/      # Conversion funnel
│   ├── billing/         # Subscription management
│   ├── codex/           # Knowledge layer UI
│   ├── voice/           # Voice calibration
│   ├── governor/        # Governance dashboard
│   └── ...              # 20+ more domain folders
├── hooks/               # 14 custom hooks
├── stores/              # Auth context, UI state (React Context)
├── api/                 # API client layer
├── lib/                 # Firebase config, utilities
└── providers/           # React context providers
```

## Middleware Stack

Three middleware layers process every request in order:

| Middleware | Purpose | Header |
|-----------|---------|--------|
| **CORSMiddleware** | Allow cross-origin requests from frontend | Standard CORS |
| **RequestIDMiddleware** | Assign unique request ID for distributed tracing | `X-Request-ID` |
| **TenantContextMiddleware** | Extract tenant identity, store in `contextvars` | `X-Tenant-ID` |

The `TierGate` decorator (not middleware) enforces subscription tier requirements on individual endpoints, returning `402 PAYMENT_REQUIRED` with an `X-Required-Tier` header when access is denied.

## Service Layer (146+ Services)

Services are grouped by domain. Each service is a plain Python class with constructor-injected dependencies (database session, LLM client, other services).

### Creative Pipeline

| Service | Purpose |
|---------|---------|
| `InterviewService` | Cold-start interview sessions (5-10 questions) |
| `BrainstormService` | Pre-project ideation with context injection (NotebookLM, Pinterest) |
| `StoryBibleService` | Extract narrative structure from interview and sources |
| `VoiceCalibrationService` | Tournament-based voice learning (3-5 rounds) |
| `DirectorService` | Scene generation with ORA (Observe-Reason-Act) loop |
| `SceneWriterService` | Prose execution for individual scenes |
| `CreativeRefinementOrchestrator` | Generic generate-critique-revise loop (Protocol-based) |
| `BookendsService` | Generate Ch1 + final chapter with setup-payoff proofs |

### Governance & Quality

| Service | Purpose |
|---------|---------|
| `GovernorService` | Kill Switches, N1-N5 violation taxonomy, ORA loop |
| `TournamentQualityValidator` | Effect size, p-value, diversity metrics |
| `VoiceStabilityValidator` | Multi-window drift detection |
| `CanonEvolutionManager` | Weighted consensus voting |
| `WildcardInjectionService` | "Move 37" creative surprises (0.2 injection ratio) |
| `StuckDetectionService` | Template contamination detection, pivot suggestions |

### Knowledge & Codex

| Service | Purpose |
|---------|---------|
| `CrystallizationService` | Layer 3 (raw) to Layer 2 (crystallized) transition |
| `ConversationCrystallizerSkill` | Extract decisions/patterns/preferences from parsed conversations |
| `KnowledgeExtractor` | Parallel LLM extraction passes with confidence filtering |
| `VesselPullService` | Pull Codex chunks into project-scoped Layer 1 |
| `RetrievalScorer` | Query-aware scoring: recency + blessing importance + embedding relevance |
| `DirectorContextService` | Assemble LLM context (Layer 1 > Layer 2 precedence) |
| `GraduationService` | Promote project learnings to sovereign Codex |
| `URLIntelligenceSkill` | Three-ring URL analysis (tenant insight, director memo, triage) |
| `InboxRouterSkill` | Classify incoming files and route to downstream services |

### Video Pipeline

| Service | Purpose |
|---------|---------|
| `VideoDirectorService` | Shot planning from scene content |
| `VideoDraftService` | Cheap-model multi-variant generation |
| `VideoSelectionService` | Creator preference capture |
| `VideoFinalService` | Premium-model final render |
| `VideoTournamentService` | Tournament runner for video selection |

### Source Ingestion

| Service | Purpose |
|---------|---------|
| `SourceImportService` | File and JSON import |
| `NotebookLMBridgeService` | NotebookLM CLI integration |
| `GmailIndexService` | Gmail thread indexing |
| `DriveIndexService` | Google Drive document indexing |
| `YoutubeRSSService` | YouTube channel ingestion |
| `PinterestOAuthService` | Pinterest profile analysis for visual style |

### Business & Operations

| Service | Purpose |
|---------|---------|
| `BillingService` (ABC) | Provider-agnostic billing interface |
| `PaddleBillingService` | Paddle Billing API v2 adapter |
| `CreditLedgerService` | Double-entry credit accounting |
| `BudgetOrchestrator` | Per-tenant budget gating and BYOK routing |
| `SurfaceAccessService` | Tier-to-surface mapping (telegram/web/api/mcp) |
| `SchedulerService` | APScheduler wrapper with PostgreSQL persistence |
| `HealthCheckService` | PostgreSQL + Neo4j + LLM provider checks |
| `CreatorProfileService` | Creator Development Engine (growth edges, provenance) |

## Sinks (18 Output Destinations)

All sinks implement a `BaseSink` interface and are registered in the `SinkRouter`. The Outbox Relay dispatches approved entries to sinks asynchronously.

| Category | Sinks |
|----------|-------|
| **Video** | Kling, Higgsfield, Veo, Seedance, Aggregator (Fal/Runware), TikTok |
| **Text** | Medium, Substack, Newsletter |
| **Publishing** | KDP Ebook (EPUB via Pandoc), ElevenLabs Audiobook (TTS M4B) |
| **Messaging** | Telegram |
| **Social** | Moltbook (monitoring) |

## API Surface

86 routers are registered under `/api/v1/`. Key endpoint groups:

| Group | Prefix | Purpose |
|-------|--------|---------|
| **Auth** | `/auth`, `/auth-providers` | Login, OAuth, BYOK key storage |
| **Projects** | `/projects` | CRUD, format selection, research upload |
| **Creative** | `/brainstorm`, `/bookends`, `/interview`, `/voice-calibration` | Ideation and calibration |
| **Generation** | `/director`, `/scenes`, `/video`, `/video-drafts` | Content generation |
| **Knowledge** | `/codex`, `/vessel`, `/canon`, `/crystallizer` | Knowledge stack |
| **Governance** | `/governor`, `/review` | Kill switch audit, quality review |
| **Business** | `/billing`, `/budget`, `/pricing`, `/beta`, `/costs` | Subscriptions and credits |
| **Operational** | `/health`, `/scheduler`, `/signals`, `/council` | Monitoring and intelligence |
| **Admin** | `/admin`, `/architect`, `/telegram-admin` | Director tools |

## Data Stores

### PostgreSQL (Primary)

- **Role:** Relational data, user state, event sourcing, outbox
- **Driver:** asyncpg (async) + psycopg2 (sync fallback)
- **Pool:** size=20, max_overflow=10
- **Tenant isolation:** Schema-per-tenant (`tenant_{id}` PostgreSQL schemas)
- **Extensions:** pgvector (embedding storage)
- **Migrations:** Alembic (`backend/migrations/versions/*`), pre-deploy migration runner with startup fallback path

### Neo4j 5 (Knowledge Graph)

- **Role:** Entity relationships, contradiction detection, GraphRAG queries
- **Driver:** neo4j async driver
- **Plugin:** APOC
- **Tenant isolation:** Cypher query injection (`inject_tenant_filter()`)
- **Status:** Optional — app starts without Neo4j (wrapped in try/except)

## Infrastructure

### Docker Compose (Local Development)

```yaml
services:
  api:       # FastAPI backend
  postgres:  # pgvector/pg16 with health checks
  neo4j:     # Neo4j 5 Community with APOC
```

### Cloud Run (Production)

- **Backend:** `ces-backend-331630083873.us-central1.run.app`
- **Frontend:** Firebase Hosting (`app.writersroom.online`)
- **Startup:** startup migration path is now fallback-only; primary migration path is pre-deploy runner
- **Scale:** Scale-to-zero; migration execution expected in deploy stage, not per cold start

### CI/CD

- **Workflow:** `.github/workflows/ci.yml`
- **Lint:** Ruff format check + Ruff linting + LLM import guardrail
- **Test:** `pytest tests/ -v --tb=short` (in-memory SQLite)
- **Deploy:** Push to `main` triggers automatic deployment

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Reasoning** | Society of Thought (tournaments) | Multi-variant generation + selection outperforms single-shot prompting (SOT-001: 40% peak capture) |
| **Governance** | Narrative Governor (N1-N5 taxonomy) | Deterministic kill switches prevent hallucination; LLMs cannot reliably self-govern |
| **Knowledge** | Three-layer Codex (Sovereign > Project > External) | Director-blessed knowledge takes precedence; trust tiers prevent contamination |
| **Consistency** | Transactional Outbox Pattern | Atomic writes across PostgreSQL + Neo4j + EventStore without 2PC |
| **LLM access** | BYOK (Bring Your Own Key) | Users supply their own API keys; platform provides the intelligence layer (soul + state + policy) |
| **Billing** | Provider-agnostic ABC | `BillingService` interface with `MockBillingService` and `PaddleBillingService` adapters |
| **Multi-tenancy** | Schema-per-tenant | PostgreSQL `search_path` isolation; Neo4j Cypher-level filtering |
| **Migration** | PORT from Writers Factory | Cherry-picked ~20K LOC from desktop MVP rather than greenfield |

## Configuration

All configuration flows through `backend/app/core/config.py` (350+ settings). Key categories:

| Category | Examples |
|----------|---------|
| **App** | `CES_APP_NAME`, `CES_ENVIRONMENT`, `CES_DEBUG` |
| **Database** | `CES_DATABASE_URL`, `CES_NEO4J_URI` |
| **Auth** | `CES_FIREBASE_PROJECT_ID`, `CES_ALLOW_DEV_AUTH`, `CES_DIRECTOR_API_KEY` |
| **LLM** | `CES_USE_MOCK_LLM`, `CES_LLM_MODEL` |
| **OAuth** | `CES_GOOGLE_CLIENT_ID`, `CES_PINTEREST_CLIENT_ID`, etc. |
| **Billing** | `CES_BILLING_PROVIDER` (mock/paddle), `CES_BILLING_WEBHOOK_SECRET` |
| **Video** | Provider-specific API keys (Kling, Higgsfield, Veo, Seedance) |
| **Features** | 20+ feature flags (`CES_SCHEDULER_ENABLED`, `CES_MEMORY_SYNTHESIS_ENABLED`, etc.) |

See [configuration.md](configuration.md) for the full environment variable reference.

## Related Documents

- [Data Model](data-model.md) — Database schema, models, relationships
- [Auth System](auth-system.md) — Firebase Auth, dev bypass, token verification
- [Multi-Tenancy](multi-tenancy.md) — Tenant isolation, schema-per-tenant
- [Event Sourcing](event-sourcing.md) — Outbox pattern, event store, relay
- [Testing](testing.md) — Test strategy, MockLLMClient, CI/CD
