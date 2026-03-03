# Data Model

> CES uses 60 SQLAlchemy models across PostgreSQL (primary store) and Neo4j (knowledge graph). Models are organized by domain: tenant/identity, creative projects, knowledge/codex, video, governance, billing, and operations. All models use UUID primary keys and automatic timestamps.

## Database Setup

**File:** `backend/app/core/database.py`

- **Async engine:** `create_async_engine()` with asyncpg driver
- **Pool:** `pool_size=20`, `max_overflow=10`
- **Session:** `async_sessionmaker` with `expire_on_commit=False`
- **Sync fallback:** Separate sync engine (psycopg2) for blocking operations
- **Tenant isolation:** Schema-per-tenant via `SET search_path` (see [multi-tenancy.md](multi-tenancy.md))

### Session Patterns

```python
# Basic session (platform-level operations)
async def endpoint(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Model).where(...))

# Tenant-isolated session (user-scoped operations)
async def endpoint(db: AsyncSession = Depends(get_db_with_tenant)):
    # search_path already set to tenant_{id}
    result = await db.execute(select(Scene).where(...))
```

## Base Classes

All models inherit from `Base` (SQLAlchemy `DeclarativeBase`) and typically include:

| Mixin | Fields | Purpose |
|-------|--------|---------|
| `UUIDMixin` | `id` (UUID, auto-generated) | Primary key |
| `TimestampMixin` | `created_at`, `updated_at` | Audit timestamps |

## Model Groups

### Multi-Tenancy & Identity

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `Tenant` | `tenants` | `tenant_id`, `display_name`, `persona_name`, `role` (TenantRole), `tier` (TenantTier), `status` (TenantStatus), `schema_name`, `subscription_tier`, `billing_customer_id`, `subscription_status` | Multi-tenant account. 6 billing columns for subscription management |
| `User` | `users` | `email`, `tenant_id`, `firebase_uid`, `display_name` | Firebase-authenticated user profiles |
| `TelegramUser` | `telegram_users` | `telegram_id`, `tenant_id`, `user_id`, `status` | Maps Telegram chat IDs to tenants |
| `VaultEntry` | `vault_entries` | `tenant_id`, `key_type` (VaultKeyType), `label`, `encrypted_value`, `is_active` | Encrypted API credentials (Fernet). 45 key types covering LLM providers, social platforms, and services |
| `CreatorPresence` | `creator_presence` | `tenant_id`, `literary_tradition`, `tone_profile`, `energy_level`, `calibration_mode`, `confidence_score` | Tenant-level style DNA (1:1 with Tenant) |
| `Presence` | `presence` | `tenant_id` | Real-time status tracking |

### Creative Projects & Structure

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `Project` | `projects` | `title`, `owner_uid`, `status` (ProjectStatus), `genre`, `logline`, `framework_id`, `format` (ContentFormat), `big_idea` | Top-level story container |
| `InterviewSession` | `interview_sessions` | `project_id`, `status` (InterviewStatus), `current_question_id`, `core_answers` (JSON), `framework_id` | Interview progress tracking |
| `BrainstormSession` | `brainstorm_sessions` | `tenant_id`, `project_id`, `owner_uid`, `status` (BrainstormStatus), `structure_sketches` (JSON), `format_selection` (JSON) | Pre-project discovery conversations |
| `BrainstormMessage` | `brainstorm_messages` | `session_id`, `role`, `content`, `metadata` (JSON) | Individual messages in brainstorm |
| `Beat` | `beats` | `project_id`, `name`, `beat_type`, `description`, `act`, `sequence_order`, `graph_layer` | Story structural beats |
| `Character` | `characters` | `project_id`, `name`, `role`, `description`, `goal`, `fear`, `fatal_flaw`, `arc_start/midpoint/resolution`, `voice_notes` | Story entities |
| `Location` | `locations` | `project_id`, `name`, `description`, `significance`, `graph_layer` | Setting entities |
| `Theme` | `themes` | `project_id`, `name`, `description`, `graph_layer` | Thematic concepts |
| `Scene` | `scenes` | `project_id`, `beat_id`, `content`, `word_count`, `strategy`, `temperature`, `score`, `audit_status` (AuditStatus), `ora_iterations`, `force_accepted` | Generated scene content |
| `NarrativeRule` | `narrative_rules` | `project_id`, `rule_type`, `description`, `scope` (RuleScope), `applies_to` (JSON) | N1-N5 governance rules |

### Knowledge & Codex (Three-Layer Stack)

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `CodexChunk` | `codex_chunks` | `tenant_id`, `content`, `content_hash`, `title`, `source_type`, `trust_tier` (TrustTier), `director_blessing_score`, `voice_dna_ready`, `topic_tags` (JSON), `embedding` | Crystallized knowledge (Layer 2). Unique on `(tenant_id, content_hash)` |
| `CrystallizationEvent` | `crystallization_events` | `codex_chunk_id`, `trigger` (CrystallizationTrigger), `transformation_type`, `provenance_snapshot` (JSON) | Audit trail for knowledge crystallization |
| `ProjectVessel` | `project_vessels` | `project_id`, `tenant_id`, `codex_chunk_ids` (JSON), `pull_policy` (JSON), `local_overrides` (JSON), `story_bible_snapshot` (JSON) | Layer 1 ephemeral workspace |
| `ResearchSource` | `research_sources` | `project_id`, `filename`, `source_type`, `storage_path`, `processing_status`, `extracted_text` | Uploaded research documents |
| `ResearchChunk` | `research_chunks` | `research_source_id`, `content`, `position`, `relevance_score`, `extracted_metadata` (JSON) | Parsed chunks from research |
| `SourceConnection` | `source_connections` | `tenant_id`, `source_type`, `auth_token`, `metadata` (JSON) | OAuth source connectors (Gmail, Drive, YouTube, Pinterest) |
| `SourceIndex` | `source_indexes` | `source_connection_id`, `item_id`, `title`, `content`, `indexed_at` | Indexed items from sources |
| `Exemplar` | `exemplars` | `project_id`, `content`, `source`, `category`, `embedding`, `notes` | Curated quality exemplars for Canon |

### Voice & Style

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `VoiceProfile` | `voice_profiles` | `project_id`, `pov`, `tense`, `voice_type`, `sentence_rhythm`, `vocabulary_level`, `characteristic_phrases` (JSON), `anti_patterns` (JSON), `tournament_data` (JSON) | Voice calibration results |
| `VisualExemplar` | `visual_exemplars` | `project_id`, `scene_type`, `mood`, `description`, `video_url`, `frame_position`, `quality_score`, `style_tags` (JSON) | Video frame references for visual consistency |

### Video Pipeline

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `VideoDraftRun` | `video_draft_runs` | `scene_id`, `shot_id`, `text_prompt`, `num_variants`, `provider`, `model`, `status` (DraftRunStatus), `variants` (JSON), `total_cost_usd` | Cheap-model variant generation |
| `VideoPreferenceSignal` | `video_preference_signals` | `draft_run_id`, `winner_variant_id`, `score_aesthetic`, `score_clarity`, `score_character_match` | Creator preference recording |
| `VideoFinalRun` | `video_final_runs` | `scene_id`, `shot_id`, `text_prompt`, `camera_instructions`, `provider`, `model`, `status`, `video_url`, `actual_cost_usd` | Premium-model final render |
| `ShotPlan` | `shot_plans` | `scene_id`, `shot_type`, `description`, `duration_seconds`, `visual_style` (JSON) | Shot-level planning |
| `RenderJob` | `render_jobs` | `video_final_run_id`, `external_id`, `status`, `provider_response` (JSON) | External render service job tracking |

### Comics Mode

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `ComicSeries` | `comic_series` | `project_id`, `title`, `description`, `issue_count` | Multi-issue comic container |
| `ComicIssue` | `comic_issues` | `series_id`, `issue_number`, `page_count`, `status` | Single issue |
| `ComicPage` | `comic_pages` | `issue_id`, `page_number`, `panel_count`, `layout_description` (JSON) | Comic page |
| `ComicPanel` | `comic_panels` | `page_id`, `panel_index`, `layout_role` (PanelLayout), `beat_type` (ComicBeatType), `description`, `visual_style` (JSON) | Individual panel |
| `VisualStyleCanon` | `visual_style_canon` | `project_id`, `palette` (JSON), `line_weight`, `panel_density`, `dialogue_density` | Visual consistency rules |

### Governance & Quality

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `Canon` | `canon` | `project_id`, `name`, `description`, `exemplars` (JSON), `rules` (JSON), `track_record` (JSON) | Quality standards and rules |
| `CanonVote` | `canon_votes` | `canon_id`, `turn_number`, `variant_id`, `dimension`, `score`, `explanation` | Voting record for Canon decisions |
| `ContradictionSignal` | `contradictions` | `project_id`, `entity_type`, `signal_type`, `severity`, `description`, `related_entities` (JSON) | Flash vs Foundry contradiction tracking |
| `DismissedFinding` | `dismissed_findings` | `tenant_id`, `fingerprint`, `dismissed_at`, `expires_at` | Observation dismissals (30-day auto-expiry) |

### Budget & Billing

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `BudgetAllocation` | `budget_allocations` | `tenant_id`, `monthly_credit_limit_usd` (Decimal), `credits_used_usd`, `period_start`, `period_end`, `subscription_tier` | Monthly credit budget |
| `UsageRecord` | `usage_records` | `tenant_id`, `provider`, `model`, `action_type` (ActionType), `input_tokens`, `output_tokens`, `estimated_cost_usd`, `is_byok` | Per-call LLM usage |
| `CreditTransaction` | `credit_transactions` | `tenant_id`, `transaction_type` (CreditTransactionType), `amount_usd` (Decimal), `description`, `expires_at` | Double-entry credit ledger |
| `PricingTier` | `pricing_tiers` | `tier_name`, `monthly_price_usd`, `included_credits_usd`, `features` (JSON), `limits` (JSON) | Explorer/Creator/Showrunner pricing |
| `BetaCode` | `beta_codes` | `code`, `tier`, `uses_remaining`, `is_redeemed`, `redeemed_by`, `redeemed_at` | Beta invitation codes (SHA-256 hashed) |

### Creator Development

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `CreatorProfile` | `creator_profiles` | `tenant_id`, `decisions_made`, `original_words_added`, `ai_suggestions_modified`, `growth_edges` (JSON), `provenance_score`, `tier_classification` | Development engine metrics |
| `CreativeDecisionEvent` | `creative_decision_events` | `tenant_id`, `project_id`, `decision_type` (DecisionType, 19 types), `decision_weight`, `metadata` (JSON) | Append-only creative choice log |

### Event Sourcing & Outbox

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `EntityEvent` | `entity_events` | `entity_type`, `entity_id`, `project_id`, `event_type`, `initial_state` (JSON), `changes` (JSON), `full_state` (JSON), `ai_generated` | Immutable append-only event log |
| `OutboxEntry` | `outbox_entries` | `type` (OutboxType, 24 types), `payload` (JSON), `status` (OutboxStatus), `retry_count`, `created_at`, `processed_at` | Transactional outbox for eventual consistency |

### Operations

| Model | Table | Key Fields | Purpose |
|-------|-------|-----------|---------|
| `ScheduledJob` | `scheduled_jobs` | `tenant_id`, `job_type`, `cron_expression`, `is_active`, `next_run_at` | APScheduler job metadata |
| `ScheduledJobLog` | `scheduled_job_logs` | `scheduled_job_id`, `execution_status`, `started_at`, `completed_at`, `error_message` | Job execution audit trail |
| `StandingOrder` | `standing_orders` | `tenant_id`, `order_type`, `recurrence`, `active` | Recurring reminders |
| `Task` | `tasks` | `tenant_id`, `task_type`, `description`, `status`, `assigned_at`, `completed_at` | Work item tracking |
| `DirectorSession` | `director_sessions` | `project_id`, `session_number`, `rubric_config` (JSON), `tournament_results` (JSON) | Director Mode session persistence |
| `MemorySynthesis` | `memory_synthesis` | `tenant_id`, `synthesis_type`, `content`, `confidence_score` | Crystallized patterns from sessions |
| `ModelUsageLog` | `model_usage_logs` | `provider`, `model`, `action_type`, `input_tokens`, `output_tokens`, `cost_usd` | LLM invocation metrics |
| `TrailerShare` | `trailer_shares` | `tenant_id`, `trailer_id`, `platform`, `share_url`, `views`, `signups_attributed` | Share-to-unlock tracking |
| `MediaAsset` | `media_assets` | `project_id`, `asset_type`, `storage_path`, `url` | Generated media files |

## Key Enums

### Project & Content

| Enum | Values | Used By |
|------|--------|---------|
| `ProjectStatus` | DRAFT, ACTIVE, ARCHIVED | Project |
| `ContentFormat` | NOVEL, TV_SERIES, SHORT_FORM, COMIC | Project |
| `InterviewStatus` | STARTED, IN_PROGRESS, COMPLETED, ABANDONED | InterviewSession |
| `AuditStatus` | PENDING, PASSED, FAILED_REJECTED, FAILED_ACCEPTED, ESCALATE_PARDON | Scene |
| `GraphLayer` | RESEARCH, STORY | Beat, Character, Location, Theme |

### Knowledge & Trust

| Enum | Values | Used By |
|------|--------|---------|
| `TrustTier` | SOVEREIGN, PROJECT_SCOPED, EXTERNAL | CodexChunk |
| `CrystallizationTrigger` | DIRECTOR_APPROVAL, AUTO_THRESHOLD, MANUAL | CrystallizationEvent |

### Multi-Tenancy

| Enum | Values | Used By |
|------|--------|---------|
| `TenantRole` | DIRECTOR, ARCHITECT, AGENT, OPERATOR, GUEST | Tenant |
| `TenantTier` | FREE, PRO, SOVEREIGN | Tenant |
| `TenantStatus` | PROVISIONING, ACTIVE, SUSPENDED, QUARANTINED | Tenant |

### Event Sourcing

| Enum | Values | Used By |
|------|--------|---------|
| `OutboxStatus` | PENDING, PROCESSING, APPROVED, COMPLETED, FAILED, FAILED_SINK | OutboxEntry |
| `OutboxType` | 24 types (NEO4J_UPSERT, SOCIAL_DRAFT, VIDEO_RENDER_REQUEST, etc.) | OutboxEntry |
| `EventType` | CREATED, UPDATED, DELETED, SNAPSHOT, RESTORED | EntityEvent |
| `EntityType` | PROJECT, CHARACTER, LOCATION, RULE, THEME, BEAT, SCENE, etc. | EntityEvent |

### Billing

| Enum | Values | Used By |
|------|--------|---------|
| `CreditTransactionType` | SUBSCRIPTION_GRANT, PROMOTION, PURCHASE, LLM_USAGE, VIDEO_RENDER, EXPIRATION, etc. | CreditTransaction |
| `ActionType` | BRAINSTORM, GENERATE_SCENE, POLISH, MULTIPLY, VOICE_CALIBRATE, SCORE, etc. | UsageRecord |

## Key Relationships

### Tenant-Centric Hierarchy

```
Tenant (1) ──→ VaultEntry (many)      [cascade delete]
Tenant (1) ──→ CreatorPresence (1)     [cascade delete]
Tenant (1) ──→ CodexChunk (many)       [cascade delete]
Tenant (1) ──→ BudgetAllocation (1)
Tenant (1) ──→ CreditTransaction (many)[cascade delete]
Tenant (1) ──→ Project (many)          [via owner_uid]
```

### Project Hierarchy

```
Project (1) ──→ InterviewSession (many)  [cascade]
Project (1) ──→ Beat (many)              [cascade]
Project (1) ──→ Character (many)         [cascade]
Project (1) ──→ Location (many)          [cascade]
Project (1) ──→ Theme (many)             [cascade]
Project (1) ──→ Scene (many)             [cascade]
Project (1) ──→ ResearchSource (many)    [cascade]
Project (1) ──→ VoiceProfile (many)      [cascade]
Project (1) ──→ Canon (many)             [cascade]
Project (1) ──→ ProjectVessel (1)        [cascade]
```

### Crystallization Pipeline

```
CodexChunk (1) ←── CrystallizationEvent (many)  [cascade]
```

## Migrations

52 Alembic migrations in `backend/migrations/versions/`, numbered sequentially from `001` to `052`. Migrations are auto-run on startup via `entrypoint.sh`.

**Migration Pattern:**
- Each migration is an Alembic revision file
- Forward-only in production (downgrades exist but are not recommended)
- All models auto-discovered via `import app.models` in `env.py`
- Supports both async (asyncpg) and sync (psycopg2) engines

**Recent migrations include:**
- `040` — Pricing tiers, credits, beta codes
- `042` — Creative decision events (CDE)
- `048` — Skill framework (Layer 1)
- `050` — Dismissed findings table
- `051` — Google free tier
- `052` — Tenant persona name

## Neo4j Graph Layer

Neo4j stores entity relationships as a knowledge graph, complementing PostgreSQL's relational storage.

**Node types:** Tenant, Character, Location, Rule, Theme, Beat, Scene, Entity, Ephemeral

**Relationship types:** OWNS, RELATES_TO, SOURCED_FROM, CONTAINS

**Tenant isolation:** Cypher queries are modified at runtime by `inject_tenant_filter()` to scope all operations to the current tenant's subgraph. See [multi-tenancy.md](multi-tenancy.md) for details.

## Related Documents

- [Architecture Overview](architecture-overview.md) — System architecture
- [Multi-Tenancy](multi-tenancy.md) — Schema-per-tenant isolation
- [Event Sourcing](event-sourcing.md) — Outbox pattern, event store
- [Auth System](auth-system.md) — How users get in
