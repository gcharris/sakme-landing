# Event Sourcing & Outbox Pattern

> CES uses event sourcing for creative entity history (scenes, characters, beats) and the Transactional Outbox Pattern for atomic writes across PostgreSQL, Neo4j, and external sinks. The OutboxRelay processes pending entries asynchronously with retry logic and HITL (Human-in-the-Loop) approval gates.

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/models/event.py` | `EntityEvent` model — immutable append-only event log |
| `backend/app/models/outbox_entry.py` | `OutboxEntry` model — transactional outbox with 24 entry types |
| `backend/app/services/outbox_relay.py` | Background worker that processes outbox entries |
| `backend/app/sinks/base_sink.py` | `BaseSink` interface for output destinations |
| `backend/app/services/atomic_write_manager.py` | Atomic write coordination |

## Why Event Sourcing?

Creative work needs full history. When a scene is edited, overwritten, or accepted by the Director, the previous versions must be recoverable. CES stores every mutation as an immutable event, allowing reconstruction of any entity at any point in time.

**Why the Outbox Pattern?** CES writes to three stores (PostgreSQL, Neo4j, EventStore). Two-phase commit (2PC) is fragile across heterogeneous stores. Saga patterns add complexity. The Transactional Outbox writes intent to PostgreSQL atomically with the business data, then a background relay processes the intent asynchronously. This is the simplest pattern that guarantees at-least-once delivery.

## Entity Events

**File:** `backend/app/models/event.py`

### Model: `EntityEvent`

| Field | Type | Purpose |
|-------|------|---------|
| `entity_type` | EntityType enum | What kind of entity changed |
| `entity_id` | UUID | Which entity |
| `project_id` | UUID | Parent project |
| `event_type` | EventType enum | What happened |
| `timestamp` | datetime | When it happened |
| `initial_state` | JSON | State before the change |
| `changes` | JSON list | Delta (list of field changes) |
| `full_state` | JSON | Complete state snapshot |
| `ai_generated` | bool | Was this change made by AI? |
| `event_number` | int | Sequential counter per entity |

### Entity Types

```
PROJECT, CHARACTER, LOCATION, RULE, THEME, BEAT, SCENE, VOICE_BUNDLE, STORY_BIBLE
```

### Event Types

```
CREATED, UPDATED, DELETED, SNAPSHOT, RESTORED
```

### State Reconstruction

Current state is reconstructed from the last `SNAPSHOT` event plus all subsequent `UPDATED` events. The `full_state` field on each event makes point-in-time queries efficient without full replay.

**Indexes:** `(entity_type, entity_id, timestamp)`, `(project_id, timestamp)`

## Transactional Outbox

**File:** `backend/app/models/outbox_entry.py`

### Model: `OutboxEntry`

| Field | Type | Purpose |
|-------|------|---------|
| `type` | OutboxType enum | What kind of work to do |
| `payload` | JSON | Type-specific data |
| `status` | OutboxStatus enum | Processing state |
| `retry_count` | int | Number of retry attempts |
| `error_message` | str, nullable | Last error |
| `created_at` | datetime | When the intent was recorded |
| `processed_at` | datetime, nullable | When processing completed |

### Status Lifecycle

```
PENDING → PROCESSING → COMPLETED
                    ↘ FAILED
                    ↘ FAILED_SINK

PENDING → APPROVED → PROCESSING → COMPLETED   (HITL path)
```

Entries requiring human approval (e.g., `SOCIAL_DRAFT`, `VIDEO_RENDER_REQUEST`) must move through `APPROVED` status before the relay processes them.

### Outbox Types (24)

| Category | Types |
|----------|-------|
| **Infrastructure** | `NEO4J_UPSERT`, `EVENT_APPEND` |
| **Governance** | `SOCIAL_DRAFT`, `SYNTHESIS_PACKAGE`, `REVIEW_FINDING` |
| **Creative Output** | `VIDEO_RENDER_REQUEST`, `NEWSLETTER_PUBLISH`, `VIDEO_UPLOAD`, `EBOOK_EXPORT`, `AUDIOBOOK_EXPORT` |
| **Budget** | `USAGE_RECONCILIATION` |
| **Routing** | `DIRECT_LOG`, `IMPROVEMENT_PROPOSAL`, `ARCHITECT_COMMAND`, `CI_FAILURE` |

### Payload Examples

**NEO4J_UPSERT:**
```json
{
  "query": "MERGE (c:Character {id: $id}) SET c.name = $name",
  "params": {"id": "uuid-here", "name": "Maya"}
}
```

**SOCIAL_DRAFT:**
```json
{
  "content": "Draft tweet content...",
  "target_medium": "twitter",
  "tenant_id": "uuid-here"
}
```

**VIDEO_RENDER_REQUEST:**
```json
{
  "scene_id": "uuid-here",
  "provider": "kling",
  "prompt": "A detective walks through rain...",
  "duration_seconds": 5
}
```

## Outbox Relay

**File:** `backend/app/services/outbox_relay.py`

The relay is a background worker that polls for pending outbox entries and dispatches them to their destinations.

### Processing Loop

```
┌─────────────────────────────┐
│ SELECT outbox_entries        │
│ WHERE status IN              │
│   (PENDING, APPROVED)        │
│ ORDER BY created_at          │
│ LIMIT 20                     │
│ FOR UPDATE SKIP LOCKED       │  ← Row-level locking
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ For each entry:              │
│  1. Set status = PROCESSING  │
│  2. Dispatch by type         │
│  3. On success: COMPLETED    │
│  4. On failure: retry_count++│
│     If retries >= 5: FAILED  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ If batch was full (20):      │
│   → Process next batch       │
│ Else:                        │
│   → Sleep 1 second           │
└─────────────────────────────┘
```

### Dispatch Routing

| OutboxType | Destination | Action |
|-----------|-------------|--------|
| `NEO4J_UPSERT` | Neo4j | Execute Cypher query from payload |
| `EVENT_APPEND` | Event store | Write EntityEvent |
| `DIRECT_LOG` | Neo4j | Create Ephemeral node (shadow sync) |
| `SOCIAL_DRAFT` | SinkRouter | Route to social media sink |
| `SYNTHESIS_PACKAGE` | SinkRouter | Route to content sink |
| `VIDEO_RENDER_REQUEST` | SinkRouter | Route to video provider |
| `NEWSLETTER_PUBLISH` | SinkRouter | Route to Substack/newsletter sink |
| `USAGE_RECONCILIATION` | Budget service | Replay failed budget calls |

### Sink Router Integration

The relay uses the `SinkRouter` to dispatch approved entries to registered sinks by target medium:

**Registered sinks:** Moltbook, Higgsfield, Kling, Seedance, Substack, YouTube, TikTok, KDP Ebook, ElevenLabs Audiobook

**Result handling:**

| Sink Result | Relay Action |
|------------|-------------|
| `SUCCESS` | Mark entry as `COMPLETED` |
| `RETRY` | Increment `retry_count` |
| `FAILED` | Mark as `FAILED_SINK` |

### Concurrency

`SELECT FOR UPDATE SKIP LOCKED` ensures multiple relay workers can run in parallel without processing the same entry twice. This is PostgreSQL's built-in row-level locking for queue patterns.

### HITL Approval Flow

Entries that require human approval (Director review before publishing):

1. Service creates `OutboxEntry` with status = `PENDING` and type = `SOCIAL_DRAFT`
2. Entry appears in the Director's **Move37Feed** (Command Center UI)
3. Director approves → status becomes `APPROVED`
4. Relay picks up `APPROVED` entries and dispatches to sinks

```
POST /api/v1/director/synthesis-feed              → List pending drafts
POST /api/v1/director/synthesis-feed/{id}/approve  → Approve for publishing
POST /api/v1/director/synthesis-feed/{id}/reject   → Reject
```

## Health & Monitoring

The relay exposes health metrics:

```python
# Count entries by status
stats = await get_outbox_stats()

# Health determination
health = await get_outbox_health()
# unhealthy if: stale entries (>5 min) OR pending count > 1000

# Cleanup old completed entries (7-day retention)
await cleanup_old_completed_entries()
```

## Atomic Write Manager

**File:** `backend/app/services/atomic_write_manager.py`

Coordinates atomic writes across PostgreSQL and the outbox within a single database transaction:

```python
async with atomic_write_manager.transaction(db) as txn:
    # Write business data to PostgreSQL
    db.add(scene)

    # Queue Neo4j sync (will be processed by relay)
    txn.queue_neo4j_upsert(cypher_query, params)

    # Queue event (will be processed by relay)
    txn.queue_event(entity_type, entity_id, event_type, state)

    # All writes committed atomically
```

If the transaction fails, both the business data and the outbox entries are rolled back together. The relay handles the asynchronous delivery to Neo4j and the event store.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Pattern** | Transactional Outbox | Simpler than 2PC or Saga for heterogeneous stores |
| **Polling** | `SELECT FOR UPDATE SKIP LOCKED` | PostgreSQL-native, supports concurrent workers |
| **Retry** | Max 5, exponential backoff | Handles transient failures without infinite loops |
| **HITL** | Status-based gating (PENDING → APPROVED) | No auto-publish; Director approves all external output |
| **Cleanup** | 7-day retention for completed entries | Prevents unbounded table growth |
| **Batch size** | 20 entries per poll | Balance between throughput and memory |

## Known Limitations

1. **Non-atomic triple-write** — The outbox ensures PostgreSQL + outbox atomicity, but Neo4j writes are eventually consistent (processed by relay). A relay crash after PostgreSQL commit but before Neo4j write will be retried, but there is a window of inconsistency.
2. **Event replay** — Full event replay is not yet implemented. Current state comes from `full_state` snapshots rather than replaying all events from genesis.

## Related Documents

- [Architecture Overview](architecture-overview.md) — Where event sourcing fits
- [Data Model](data-model.md) — EntityEvent and OutboxEntry models
