# Configuration

> Environment variables, defaults, and operational flags for Life OS core.

## Overview

Configuration is loaded via `pydantic-settings` in `backend/app/core/config.py` (`Settings` class). Runtime env vars use `CES_` prefix.

## Configuration Loading

```python
from app.core.config import get_settings

settings = get_settings()  # Cached singleton via @lru_cache
```

The most reliable source of truth is always `backend/app/core/config.py`; this document highlights operator-critical groups and commonly used flags.

## Core Variable Groups

### Application

| Variable | Default | Purpose |
|----------|---------|---------|
| `app_name` | `"Context Engine Studio"` | Application name |
| `app_version` | `"0.1.0"` | Application version |
| `debug` | `false` | Debug mode |
| `environment` | `"development"` | Environment name |
| `project_root` | environment-specific | Project root path |

### API

| Variable | Default | Purpose |
|----------|---------|---------|
| `api_prefix` | `"/api/v1"` | API route prefix |
| `allowed_origins` | See config.py | CORS origins list |
| `frontend_url` | deployment-specific | Frontend URL for redirects |

### Database

| Variable | Default | Purpose |
|----------|---------|---------|
| `database_url` | `postgresql+asyncpg://ces:ces@localhost:5432/ces` | PostgreSQL connection |
| `database_echo` | `false` | SQLAlchemy query logging |

### Migration Control

| Variable | Default | Purpose |
|----------|---------|---------|
| `auto_migrate_on_startup` | `true` | Enable startup migration path |
| `auto_migrate_mode` | `"blocking"` | Migration mode (`blocking` or `background`) |
| `auto_migrate_fail_fast` | `false` | Startup behavior on migration failure |
| `auto_migrate_timeout_seconds` | `300` | Background migration timeout |
| `CES_SKIP_AUTO_MIGRATE` | unset | Env override to skip startup migration path |

### Neo4j

| Variable | Default | Purpose |
|----------|---------|---------|
| `neo4j_uri` | `bolt://localhost:7687` | Neo4j connection URI |
| `neo4j_user` | `neo4j` | Neo4j username |
| `neo4j_password` | `neo4j` | Neo4j password |

### Google Cloud

| Variable | Default | Purpose |
|----------|---------|---------|
| `gcs_bucket` | `ces-research-uploads` | GCS bucket for file uploads |
| `gcp_project_id` | `""` | GCP project ID |
| `gcp_location` | `us-central1` | Vertex AI region |

### Authentication

| Variable | Default | Purpose |
|----------|---------|---------|
| `firebase_project_id` | `"kognist"` | Firebase project ID (empty = dev mode) |
| `allow_dev_auth` | `false` | Enable dev auth bypass |
| `director_api_key` | `""` | Secret key for Director API access |

### Telegram

| Variable | Default | Purpose |
|----------|---------|---------|
| `telegram_bot_token` | `None` | Bot token |
| `telegram_director_chat_id` | `None` | Director chat ID |
| `telegram_webhook_secret` | `None` | Webhook HMAC secret |
| `telegram_topic_creative` | `0` | Forum topic ID for creative messages |
| `telegram_topic_system` | `0` | Forum topic ID for system messages |
| `telegram_topic_knowledge` | `0` | Forum topic ID for knowledge messages |
| `telegram_topic_director_health` | `0` | Forum topic for health alerts |
| `telegram_topic_director_creators` | `0` | Forum topic for creator activity |

### Video Pipeline

| Variable | Default | Purpose |
|----------|---------|---------|
| `video_provider` | `"aggregator"` | Default video provider |
| `hf_credentials` | `None` | Higgsfield API key |
| `kling_access_key` | `None` | Kling API access key |
| `kling_secret_key` | `None` | Kling API secret key |
| `aggregator_api_key` | `None` | Fal.ai/Runware API key |
| `aggregator_default_model` | `"wan-2.5-lite"` | Default cheap draft model |
| `veo_default_model` | `"veo-3.1-fast"` | Default Veo model |
| `default_final_provider` | `"kling"` | Premium render provider |
| `default_final_model` | `"kling-3.0-pro"` | Premium render model |
| `seedance_enabled` | `false` | Seedance 2.0 feature flag |

### OAuth Providers

| Variable | Default | Purpose |
|----------|---------|---------|
| `google_client_id` | `""` | Google OAuth client ID |
| `google_client_secret` | `""` | Google OAuth client secret |
| `twitter_client_id` | `""` | Twitter OAuth client ID |
| `youtube_client_id` | `""` | YouTube OAuth client ID |
| `pinterest_client_id` | `""` | Pinterest OAuth client ID |

Callback URLs default to the Cloud Run backend URL.

### Agentic Runtime (Phase 1/2/3)

| Variable | Default | Purpose |
|----------|---------|---------|
| `agentic_loop_enabled` | `true` | Enable new loop engine path |
| `agentic_loop_max_attempts` | `3` | Retry attempts for overflow/provider errors |
| `agentic_loop_max_turns` | `50` | Safety cap for runaway loops |
| `agentic_loop_max_tool_result_chars` | `8000` | Tool result truncation limit |
| `agentic_loop_max_context_chars` | `100000` | Context budget |
| `agent_streaming_enabled` | `true` | Progressive streaming events |
| `agent_fallback_enabled` | `true` | Model fallback chain enablement |
| `agent_fallback_chain_json` | `""` | Optional custom fallback chain JSON |
| `subagent_enabled` | feature-flagged | Parent agent sub-agent delegation |
| `subagent_user_enabled` | feature-flagged | User-agent sub-agent delegation |
| `transcript_persistence_enabled` | feature-flagged | Persistent transcript writes |
| `memory_search_enabled` | feature-flagged | Past conversation search tools |

### Billing

| Variable | Default | Purpose |
|----------|---------|---------|
| `billing_provider` | `"mock"` | Billing adapter (mock/stripe/paddle) |
| `billing_webhook_secret` | `""` | Webhook signature secret |
| `paddle_api_key` | `None` | Paddle API bearer token |
| `paddle_environment` | `"sandbox"` | Paddle environment |

### Budget Orchestrator

| Variable | Default | Purpose |
|----------|---------|---------|
| `budget_orchestrator_enabled` | `false` | Enable budget gating |
| `budget_default_explorer_limit_usd` | `1.00` | Monthly limit for Explorer tier |
| `budget_default_creator_limit_usd` | `5.00` | Monthly limit for Creator tier |
| `budget_default_showrunner_limit_usd` | `15.00` | Monthly limit for Showrunner tier |
| `budget_overage_warning_threshold` | `0.8` | Warning at 80% usage |

### Architect / Agent

| Variable | Default | Purpose |
|----------|---------|---------|
| `architect_two_model_enabled` | `true` | Enable two-model agent |
| `architect_max_tool_rounds` | `1` | Tool rounds per turn |
| `architect_model` | `"gemini-2.0-flash-001"` | Architect LLM model |
| `director_mode` | `false` | Enable filesystem/shell tools |
| `director_workspace` | `"/opt/ces"` | Workspace path |

### Scheduler

| Variable | Default | Purpose |
|----------|---------|---------|
| `scheduler_enabled` | `false` | Enable APScheduler |
| `scheduler_misfire_grace_time` | `300` | Grace seconds for missed jobs |

## Feature Flags

Many features are behind explicit booleans. Several newer runtime flags default to `true` (notably agentic loop and streaming). Always verify defaults in `config.py` before rollout changes.

| Flag | Purpose |
|------|---------|
| `scheduler_enabled` | APScheduler infrastructure |
| `budget_orchestrator_enabled` | Budget-gated LLM routing |
| `standing_orders_enabled` | Recurring task system |
| `proactive_notifications_enabled` | Stuck creator alerts |
| `gmail_sync_enabled` | Gmail source sync |
| `calendar_sync_enabled` | Calendar sync |
| `director_crm_enabled` | Creator relationship management |
| `cost_dashboard_enabled` | Cost analytics |
| `ai_council_enabled` | Nightly AI debate |
| `signal_aggregation_enabled` | Business signal aggregation |
| `chain_executor_enabled` | Multi-step task chains |
| `code_agent_enabled` | Model-agnostic code agent |
| `seedance_enabled` | Seedance video sink |
| `vertex_audio_enabled` | Vertex AI audio processing |
| `claude_proxy_enabled` | Claude CLI proxy routing |

## Related Documents

- [deployment.md](deployment.md) -- How these variables are set in production
- [byok-model.md](byok-model.md) -- BYOK-specific configuration
- [auth-system.md](auth-system.md) -- Authentication configuration

## Operator Notes

- Do not commit production credential values into docs or repo `.env` files.
- Keep deployment-specific values in environment/secret manager systems.
- For front-end hosting variants (`app.agent-c.agency`, legacy domains), align `allowed_origins` + `frontend_url` per environment.
