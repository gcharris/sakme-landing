# Sovereign Codex

> Three-layer knowledge stack, crystallization, trust tiers, and graduation.

## Overview

The Sovereign Codex is CES's owned knowledge base. It implements a **three-layer knowledge architecture** where raw external data is progressively refined into trusted, training-ready knowledge. The key principle: knowledge earns trust through **Director involvement**, quantified by a "blessing score."

```
Layer 3: External (Query)     ─── Search results, API responses
    │                              Not owned, not trusted
    ▼  [Crystallization]
Layer 2: Codex (Owned)        ─── CodexChunks with trust tiers
    │                              Owned, training-ready
    ▼  [Vessel Pull]
Layer 1: Vessel (Project)     ─── Project-scoped working set
                                   Curated for current project
```

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/models/codex.py` | CodexChunk, CrystallizationEvent, enums |
| `backend/app/services/crystallization_service.py` | Layer 3 to Layer 2 transition |
| `backend/app/services/graduation_service.py` | Project outputs back to Layer 2 |
| `backend/app/services/vessel_pull_service.py` | Layer 2 to Layer 1 (project pull) |
| `backend/app/services/director_context_service.py` | Assembles LLM context from Codex |

## Trust Tiers

Defined in `TrustTier` enum in `backend/app/models/codex.py`:

| Tier | Name | Blessing Score | Source | Voice DNA Ready |
|------|------|---------------|--------|-----------------|
| **1** | `SOVEREIGN` | 1.0 | Director's own thinking | Always true |
| **2** | `PROJECT_SCOPED` | Varies | Crystallized for a project | Depends on edits |
| **3** | `EXTERNAL` | 0.0 | Raw query results | Never |

## CodexChunk Model

**File:** `backend/app/models/codex.py`

The core knowledge unit:

```python
class CodexChunk(Base, UUIDMixin, TimestampMixin):
    __tablename__ = "codex_chunks"

    tenant_id: UUID           # Owner
    content: str              # The knowledge content
    content_hash: str         # SHA-256 for deduplication
    source_type: str          # "roam", "obsidian", "twitter", etc.
    source_uri: str           # Provenance URI
    trust_tier: TrustTier     # SOVEREIGN / PROJECT_SCOPED / EXTERNAL
    director_blessing_score: float  # 0.0 to 1.0
    voice_dna_ready: bool     # Ready for voice training
    emotional_context: str    # Optional emotional tagging
    topic_tags: list          # JSON topic tags
    embedding: list           # Vector for RAG queries
```

Unique constraint on `(tenant_id, content_hash)` prevents duplicate knowledge across the same tenant.

## Crystallization

**File:** `backend/app/services/crystallization_service.py`

Crystallization is the transition from Layer 3 (external) to Layer 2 (owned). It calculates the **blessing score** based on how much the Director edited the AI-generated content:

```python
def calculate_blessing_score(agent_draft, final_content) -> tuple[float, bool]:
    """
    Returns (blessing_score, auto_voice_dna_ready).
    - High deletion = high involvement, NOT auto-ready for training
    - High addition = high involvement AND likely ready for training
    - Fully human-written (agent_draft=None) = blessing 1.0, voice ready
    """
```

### CrystallizationService

```python
class CrystallizationService:
    async def crystallize(self, tenant_id, candidate, final_content, trigger, ...) -> (CodexChunk, CrystallizationEvent)
    async def crystallize_batch(self, tenant_id, candidates, trigger, ...) -> list[tuple]
    async def sovereign_sync(self, tenant_id, source_type, content, source_uri) -> CodexChunk
```

`sovereign_sync()` is for auto-crystallizing the Director's own thinking (blessing=1.0, voice_dna_ready=True). Used by PKMS connectors (Roam, Obsidian).

### Deduplication

Before creating a new chunk, crystallization checks for existing chunks with the same `content_hash`:

```python
async def _find_existing_chunk(self, tenant_id, content_hash) -> CodexChunk | None
```

If a duplicate exists, the crystallization event is still recorded (for provenance tracking) but no new chunk is created.

## Crystallization Triggers

| Trigger | When |
|---------|------|
| `MANUAL_BLESSING` | Director explicitly approves content |
| `PROJECT_PULL` | Bulk pull into a project vessel |
| `CLUSTER_APPROVAL` | Director approves a content cluster |
| `SOVEREIGN_SYNC` | PKMS auto-sync (Roam, Obsidian) |
| `PROJECT_GRADUATION` | Project outputs graduate back to Codex |

## Transformation Types

Content can be transformed during crystallization:

| Type | Description |
|------|-------------|
| `RAW_IMPORT` | No transformation, verbatim |
| `SUMMARY` | Extractive summary (first + last sentences) |
| `VOICE_TUNED` | Rewritten in Director's voice |
| `TRIPLET_EXTRACTION` | Knowledge triplets extracted |
| `LLM_EXTRACTION` | LLM-powered extraction |
| `MERGE` | Multiple sources merged |

Note: `SUMMARY`, `VOICE_TUNED`, and `TRIPLET_EXTRACTION` currently have stub implementations that will call LLMs in production.

## Graduation

**File:** `backend/app/services/graduation_service.py`

When a project completes, its valuable work "graduates" back to the Codex as Tier 1 (SOVEREIGN) content. This closes the knowledge loop:

```
Layer 3 → Layer 2 → Layer 1 → Outputs → [Graduate] → Layer 2
```

```python
class GraduationService:
    async def graduate_project(self, project_id, policy) -> GraduationResult
    async def archive_without_graduating(self, project_id) -> None
    async def preview_graduation(self, project_id, policy) -> GraduationPreview
```

### Graduation Policy

```python
class GraduationPolicy(BaseModel):
    include_local_overrides: bool = True
    only_promoted_overrides: bool = False  # Only graduate overrides marked "promote_to_codex"
    include_outputs: bool = True
    only_approved_outputs: bool = True
```

Graduated content gets `blessing_score=1.0` and `voice_dna_ready=True` because it survived the full creative process.

After graduation, `CreatorProfileService.compute_incremental()` is triggered to update the creator's development profile.

## CrystallizationEvent Audit Trail

Every crystallization is recorded:

```python
class CrystallizationEvent(Base, UUIDMixin, TimestampMixin):
    tenant_id: UUID
    trigger: CrystallizationTrigger
    triggered_at: datetime
    triggered_by: str
    source_type: str
    source_uri: str
    source_content_hash: str
    transformation: TransformationType
    codex_chunk_id: UUID
    calculated_blessing: float
    agent_draft: str | None
    project_id: UUID | None
    notes: str | None
```

## Decision Event Integration

Crystallization emits `CreativeDecisionEvent` entries for the Creator Development Engine:

- Heavy edits (blessing < 0.5) emit `CODEX_HEAVY_EDIT` with `DEFINING` weight
- Light edits emit `CODEX_COMMIT` with `SUBSTANTIVE` weight

## Related Documents

- [learning-engine.md](learning-engine.md) -- How Layer 3 content is created
- [knowledge-graph.md](knowledge-graph.md) -- Graph projection of Codex entities
- [data-model.md](data-model.md) -- Full model reference
