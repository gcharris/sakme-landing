# Creator Development Engine (CDE) — Developer Guide

> **Last Updated:** February 27, 2026
> **Status:** CDE-1 through CDE-4 COMPLETE (production). CDE-5 deferred. SPEC-1/2/3 proposed.
> **LOC:** ~1,620 | **Tests:** ~139
> **Design Spec:** `docs/plans/2026-02-14-creator-development-engine.md`

---

## What CDE Is (and What It Is Not)

CDE is the infrastructure that learns *who a creator is becoming* by observing their consequential creative decisions over time. It produces a **creative portrait** — a warm, specific description of someone's creative identity — not an analytics dashboard.

The distinction matters. "Your ai_divergence_rate is 0.68" is analytics. "You consistently trust your own taste over the AI's — two-thirds of the time, you chose differently from what the system recommended" is a portrait. CDE produces the second kind.

CDE does NOT track data changes (that's EntityEvent). It does NOT measure quality (that's the Governor). It does NOT judge whether high or low AI usage is "good." It captures the moments where a human exercised creative judgment, then finds patterns in those moments that the human couldn't see themselves.

**Why it exists:** Every creator working with AI faces a question of creative agency. Am I directing this, or is it directing me? CDE answers that question with evidence, not feeling. It also gives the Alter Ego the data it needs to adapt its behavior to each creator's style — the foundation for Adaptive Autonomy.

---

## Architecture Overview

CDE follows a simple pipeline:

```
Services emit events → Events accumulate → Profile computed from events → Surfaces display profile
```

In detail:

```
┌───────────────────────────────────────────────────────┐
│                 EMISSION LAYER                         │
│  GovernorService, CrystallizationService,             │
│  VoiceCalibrationService, RefinementOrchestrator,     │
│  BrainstormService                                    │
│                    │                                   │
│            DecisionEmitter.emit()                     │
│            (same transaction)                         │
└────────────────────┬──────────────────────────────────┘
                     ▼
┌───────────────────────────────────────────────────────┐
│              STORAGE LAYER                             │
│  creative_decision_events table                        │
│  (append-only, never updated, never deleted)          │
└────────────────────┬──────────────────────────────────┘
                     ▼
┌───────────────────────────────────────────────────────┐
│            COMPUTATION LAYER                           │
│  CreatorProfileService.compute_profile()              │
│  ProvenanceScoreService.score_project()               │
│  ProvenanceScoreService.score_body_of_work()          │
└────────────────────┬──────────────────────────────────┘
                     ▼
┌───────────────────────────────────────────────────────┐
│              SURFACE LAYER                             │
│  REST: GET /creator/profile, /provenance, /mirror     │
│  Telegram: /mirror, /provenance                       │
│  Frontend: CreatorProfileCard, ProvenanceScoreCard,   │
│            MirrorReflection, GrowthEdgesList          │
│  Scheduler: weekly mirror recomputation               │
└───────────────────────────────────────────────────────┘
```

---

## Layer 1: The Event Model

**File:** `backend/app/models/creative_decision.py`

### CreativeDecisionEvent

The foundation. An append-only record of every moment where a creator exercised creative judgment. These are NEVER updated or deleted — they're historical facts.

**Fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `id` | UUID | Primary key |
| `tenant_id` | UUID (FK → tenants) | Who made the decision |
| `project_id` | UUID (FK → projects), nullable | Which project (null = cross-project) |
| `created_at` | DateTime (tz) | When |
| `decision_type` | String | What category (see DecisionType enum) |
| `decision_weight` | String | ROUTINE / SUBSTANTIVE / DEFINING |
| `context` | JSON | **The valuable part** — rich context about what happened |
| `ai_suggestion_divergence` | Float, nullable | 0.0 (accepted AI) to 1.0 (fully diverged) |
| `time_to_decide_seconds` | Float, nullable | How long the creator deliberated |
| `ai_suggestion_snapshot` | JSON, nullable | What the AI suggested (CDE-6 delta capture) |
| `human_delta_summary` | Text, nullable | How the human changed it (CDE-6 delta capture) |

**Indexes:** Three composite indexes optimize the most common queries: (tenant_id, created_at) for profile computation, (tenant_id, decision_type) for category filtering, (project_id, created_at) for per-project scoring.

### DecisionType Enum (28+ values)

Decisions are organized into eight categories:

**Tournament decisions** — what the creator chose when presented with AI-generated variants:
- `TOURNAMENT_SELECTION` — picked a variant from the tournament
- `TOURNAMENT_OUTLIER_CHOSEN` — specifically chose the wildcard/unexpected variant
- `TOURNAMENT_OUTLIER_REJECTED` — saw the wildcard but picked something safer

**Governor decisions** — how the creator responded to governance violations:
- `GOVERNOR_OVERRIDE` — the Governor flagged a problem, creator said "I know, keep it"
- `GOVERNOR_PARDON` — acknowledged the violation but accepted anyway
- `GOVERNOR_COMPLIED` — accepted the Governor's correction
- `GOVERNOR_N6_VIOLATION` — triggered the custom N6 rule

**Crystallization decisions** — how the creator interacted with the knowledge base:
- `CODEX_COMMIT` — committed a piece to the Sovereign Codex
- `CODEX_HEAVY_EDIT` — committed but rewrote significantly first
- `CODEX_REJECTION` — the system suggested crystallizing, creator said no

**Refinement decisions** — how the creator steered the revision loop:
- `REFINEMENT_DIRECTION` — gave specific instructions for revision
- `REFINEMENT_ACCEPTED` — accepted a refinement without steering
- `REFINEMENT_OVERRIDE` — rejected the refinement entirely

**Voice decisions** — how the creator managed their creative voice:
- `VOICE_DRIFT_ACKNOWLEDGED` — noticed voice was drifting, accepted it
- `VOICE_DRIFT_CORRECTED` — noticed drift, corrected back

**Structural decisions** — large creative choices about the work itself:
- `BEAT_REORDER` — rearranged the story structure
- `FORMAT_PIVOT` — changed the output format mid-project
- `BIG_IDEA_REVISION` — changed the central thesis/theme

**Content contribution** — direct creative input from the human:
- `ORIGINAL_CONTENT_ADDED` — wrote original words (not AI-generated)
- `SOURCE_MATERIAL_CONTRIBUTED` — brought external source material

**Taxonomy graduation** — progression through the Skill→Talent→Habit→Practice lifecycle:
- `SKILL_PROMOTED_TO_TALENT` — a capability graduated from generic to personalized
- `HABIT_PRACTICED` — used a Habit (automated recurring capability)
- `TALENT_PUBLISHED` — published a Talent to the marketplace
- `TALENT_INSTALLED` — installed someone else's Talent

**Community/Gallery decisions** — social and publishing actions:
- `COMMUNITY_FOLLOW`, `COMMUNITY_LIKE`, `COMMUNITY_COMMENT`, `COMMUNITY_SHARE` — social engagement
- `GALLERY_PUBLISH`, `GALLERY_FEATURE` — showcasing work
- `EXPORT_DISPATCH` — exported to an external sink

### DecisionWeight Enum

Not all decisions carry equal weight. The weight affects how CDE computes deliberation time and provenance:

| Weight | Examples | Deliberation Multiplier |
|--------|----------|------------------------|
| `ROUTINE` | Formatting choices, minor edits | 0.5× |
| `SUBSTANTIVE` | Tournament selections, refinement steering | 1.0× (default) |
| `DEFINING` | Governor overrides, big idea revisions, format pivots | 1.5× |

A creator who spends 45 seconds on a defining decision (overriding the Governor with documented reasoning) demonstrates more deliberation than one who spends 45 seconds on a routine decision (accepting a formatting suggestion).

### The Context Field

The `context` JSON field is what makes CDE valuable. A minimal context like `{"decision": "accepted"}` is almost worthless. A rich context like this is what powers the entire downstream analysis:

```json
{
  "tournament_id": "abc-123",
  "variants_shown": 3,
  "ai_top_pick": "variant_2",
  "human_chose": "variant_1",
  "human_chose_ai_top": false,
  "variant_1_score": 0.72,
  "variant_2_score": 0.89,
  "variant_3_score": 0.65,
  "selection_reason": "The lower-scored variant had more authentic voice",
  "word_count": 1240,
  "phase": "generation"
}
```

Context is what enables growth edge detection ("you always pick the AI's top choice"), voice trajectory tracking ("your divergence is increasing across projects"), and identity statement generation ("you consistently value authenticity over polish").

---

## Layer 2: The Decision Emitter

**File:** `backend/app/services/decision_emitter.py`

### DecisionEmitter

A lightweight helper that services use to record decisions. The critical design choice: events are emitted **in the same database transaction** as the parent operation. If a Governor audit rolls back, the decision event rolls back with it. This guarantees consistency without a separate outbox.

**Usage pattern:**

```python
# Inside GovernorService.audit_scene():
emitter = DecisionEmitter(db=self.db)
await emitter.emit(
    tenant_id=tenant_id,
    decision_type=DecisionType.GOVERNOR_OVERRIDE,
    context={
        "violation_code": "N1",
        "violation_description": "Character voice contamination",
        "force_accept_reason": "Intentional stylistic choice for dream sequence",
        "severity": "HIGH",
        "scene_id": str(scene_id),
    },
    project_id=project_id,
    decision_weight=DecisionWeight.DEFINING,
    ai_suggestion_divergence=0.8,
    time_to_decide_seconds=32.5,
)
```

**Key design decisions:**

1. **Fire-and-forget safety.** The emitter wraps all non-essential logic in try/except. If persona enrichment fails, the parent operation still succeeds. The event itself is part of the transaction, but the enrichment trigger is defensive.

2. **Persona enrichment trigger.** Every 50 decisions for a tenant, the emitter calls `CreatorPersonaService.enrich_from_decisions()`. This is a lazy import to avoid circular dependencies. The persona fields on CreatorProfile (creator_type, communication_style, creative_posture, etc.) get updated based on accumulated decision patterns.

3. **No flush/commit.** The emitter adds the event to the session but never flushes or commits. The parent service's transaction handles that. This means you can emit multiple events in one transaction and they'll all commit or roll back together.

**Services that emit decisions (current integration points):**

| Service | Decision Types Emitted |
|---------|----------------------|
| GovernorService | GOVERNOR_OVERRIDE, GOVERNOR_PARDON, GOVERNOR_COMPLIED |
| CrystallizationService | CODEX_COMMIT, CODEX_HEAVY_EDIT, CODEX_REJECTION |
| VoiceCalibrationService | VOICE_DRIFT_ACKNOWLEDGED, VOICE_DRIFT_CORRECTED |
| CreativeRefinementOrchestrator | REFINEMENT_DIRECTION, REFINEMENT_ACCEPTED, REFINEMENT_OVERRIDE |
| BrainstormService | TOURNAMENT_SELECTION, TOURNAMENT_OUTLIER_CHOSEN |

---

## Layer 3: The Creator Profile

**File:** `backend/app/models/creator_profile.py`

### CreatorProfile

A computed view — one per tenant — derived from aggregated decision events. This is NOT source data. The events are the source of truth. The profile is a cache that gets recomputed periodically.

**Metric categories:**

**Taste Signature** — what does this creator's aesthetic instinct look like?
- `ai_divergence_rate` (Float) — mean of ai_suggestion_divergence across all events. Higher = creator frequently disagrees with AI recommendations.
- `outlier_affinity` (Float) — how often the creator picks unexpected options in tournaments. Measures willingness to choose the unconventional variant.

**Agency Metrics** — how much creative control does this creator exercise?
- `original_content_ratio` (Float) — proportion of total word count that's human-written (ORIGINAL_CONTENT_ADDED events vs total).
- `codex_contribution_rate` (Float) — proportion of Codex commits that received high Director blessing (score > 0.7). Measures investment in knowledge curation.
- `average_blessing_score` (Float) — mean blessing score across all Codex commits. Higher = more editorial investment.
- `governor_override_rate` (Float) — proportion of governance encounters where creator overrode or pardoned (vs complied). Measures independence from governance rules.

**Deliberation Patterns** — how does this creator make decisions?
- `average_decision_time_seconds` (Float) — mean time-to-decide, capped at 300 seconds to exclude abandoned sessions.
- `decision_density_per_project` (Float) — decisions per unique project. Higher = more engaged creator.

**Growth Edges** (JSON) — up to 5 detected patterns where the creator is plateauing or defaulting. Each has a dimension name, description, evidence count, suggested challenge, and detection timestamp. See the Growth Edge Detection section below.

**Voice Trajectory** (JSON) — per-project snapshots tracking how the creator's voice evolves. Each snapshot records decision count, average divergence, whether voice drift was acknowledged, and drift from the previous project. This reveals whether a creator is developing a stronger independent voice over time.

**Override Patterns** (JSON) — per-violation-code records of how the creator responds to each governance rule. If a creator always overrides N1 (character contamination) but always complies with N3 (arc violations), that's a meaningful signature.

**Identity Statement** (Text) — an LLM-generated 3-5 sentence creative portrait. Written in second person, warm and specific, referencing actual metrics with context. Generated when 50+ events exist and an LLM client is available; falls back to a template otherwise.

**Persona Fields (TASK-A07 extension):**
- `creator_type` — "novelist", "memoirist", "vibe_coder", "tiktok_creator", etc.
- `communication_style` — "concise", "exploratory", "directive", "collaborative"
- `creative_posture` — "collaborative", "directive", "autonomous"
- `model_affinity` — JSON mapping model names to usage proportions
- `tool_preferences` — JSON mapping tool categories to usage levels
- `onboarding_signals` — raw signals captured during the onboarding flow
- `persona_prompt` — ~200 character injection for system prompts (how the Alter Ego should talk to this creator)
- `persona_version` — increments on each recomputation

**Transient properties** (computed at read time, not stored):
- `output_diversification_score` — 0.0-1.0 measuring variety of output types (formats × sinks × pipelines)
- `output_diversification_breakdown` — dimension counts feeding the score
- `ancestry_context` — genealogy-derived identity context (FamilySearch opt-in only)

---

## Layer 4: Profile Computation

**File:** `backend/app/services/creator_profile_service.py` (~1,005 lines)

### CreatorProfileService

The intelligence layer that turns raw events into meaningful behavioral profiles. Called in three contexts:

1. **Project graduation** — GraduationService triggers incremental update
2. **Weekly scheduler** — cron job recomputes all active profiles
3. **On demand** — Telegram `/mirror` or REST `POST /creator/profile/compute`

### Constants (Tuning Knobs)

```python
MINIMUM_EVENTS_FOR_PROFILE = 20      # Don't compute until enough data
MINIMUM_EVENTS_FOR_IDENTITY = 50     # Don't generate identity statement until rich enough
MINIMUM_EVIDENCE_FOR_GROWTH_EDGE = 5 # Each edge needs minimum evidence
MAX_GROWTH_EDGES = 5                 # Cap to keep it actionable
DECISION_TIME_CAP_SECONDS = 300.0    # Exclude abandoned sessions
INCREMENTAL_GROWTH_EDGE_THRESHOLD = 10  # Redetect edges after this many new events
INCREMENTAL_IDENTITY_THRESHOLD = 50     # Regenerate identity after this many new events
```

### compute_profile(tenant_id) — Full Recomputation

Idempotent: same events produce same profile. Fetches ALL events for the tenant, computes every metric category, detects growth edges, builds voice trajectory, maps override patterns, and optionally generates an identity statement.

Returns early with counts-only if fewer than 20 events exist.

### compute_incremental(tenant_id) — Optimization

Uses count-based detection (not timestamp-based) to determine if new events have arrived since last computation. If the profile doesn't exist yet, falls back to full compute. Only redetects growth edges if 10+ new events accumulated. Only regenerates identity statement if 50+ new events accumulated. Everything else recomputes from all events (the aggregation functions are cheap since events fit in memory).

### Growth Edge Detection

The five growth edge detectors each look for specific behavioral patterns. They require minimum evidence (5 events) and are sorted by evidence count (most impactful first), capped at 5.

**1. `creative_boldness`** — Detects creators who consistently pick the AI's top recommendation.

Triggers when: `outlier_affinity < 0.15` AND `ai_divergence_rate < 0.3` AND tournament event count ≥ 5.

Suggested challenge: "Try picking a variant that surprises you next time — the unexpected choice often leads to your most distinctive work."

**2. `governance_instinct`** — Detects asymmetric governance behavior.

Triggers when: the creator overrides some violation codes >70% of the time but rarely challenges others (<20%). Requires at least 5 events per violation code.

Suggested challenge: References the specific codes — "Your confidence with N1 suggests you know when rules should bend."

**3. `editorial_investment`** — Detects declining engagement with the knowledge base.

Triggers when: average blessing score in the first half of Codex events is 0.15+ higher than the second half. Requires at least 10 Codex events (2× the minimum evidence threshold).

Suggested challenge: "Is that intentional? If so, great. If not, try giving one codex entry a heavy edit this week."

**4. `execution_authorship`** — Detects creators who contribute ideas but defer to AI for execution.

Triggers when: ORIGINAL_CONTENT_ADDED events appear in brainstorm/discovery/research phases but not in generation/writing/production phases. Requires at least 5 original content events.

Suggested challenge: "Try writing a scene yourself first, then let the AI refine it."

**5. `refinement_direction`** — Detects passive acceptance of refinements.

Triggers when: accepted refinements outnumber steered refinements by more than 3:1, OR when all refinements were accepted with zero steering. Requires at least 5 refinement events total.

Suggested challenge: "Try giving the system specific revision instructions instead of accepting the first refinement."

### Voice Trajectory

Groups events by project (sorted chronologically), computes per-project snapshots with average divergence, voice drift acknowledgment, average blessing, and drift from the previous project's divergence. This creates a longitudinal view of how the creator's independent voice is developing.

### Override Patterns

Maps governance events by violation code, counting overrides vs complies and finding the most common reason for overriding. This reveals which rules the creator considers flexible ("I know better than the Governor here") vs which they respect ("the Governor is right about this").

### Identity Statement Generation

Uses an LLM (when available) with a carefully crafted prompt that enforces: second person, warm and observant, specific metrics with context, distinctive qualities not deficiencies, invitations not criticisms, 3-5 sentences, no bullet points, no "you're doing great," no clinical language. Falls back to a template when no LLM is available.

---

## Layer 5: Provenance Scoring

**File:** `backend/app/services/provenance_score_service.py` (~398 lines)

### ProvenanceScoreService

Quantifies creative agency — NOT quality. A low score doesn't mean bad work. It means the creator used more AI autopilot. The score exists so creators can choose to demonstrate their agency when it matters to them.

Can score a single project (`score_project`) or the entire body of work (`score_body_of_work`).

### Six Components

Each scored 0.0 to 1.0:

| Component | Weight | What It Measures |
|-----------|--------|-----------------|
| `decision_density` | 0.20 | Decisions per 1,000 words (max 5.0 for normalization). More decisions = more engaged. |
| `divergence_depth` | 0.30 | Blend of scalar divergence (50%), delta documentation rate (20%), and snapshot richness (30%). How much and how well-documented the human diverged from AI. |
| `deliberation_time` | 0.10 | Weight-normalized time-to-decide. Defining decisions × 1.5, routine × 0.5. Normalized against 60-second max. |
| `original_contribution` | 0.25 | Proportion of words that are human-written. |
| `governor_engagement` | 0.15 | Proportion of all decisions that involved governance encounters. Active governance engagement signals awareness. |
| `snapshot_richness` | — | Sub-component of divergence_depth. Not weighted separately. Measures how well the AI→Human chain is documented. |

### Snapshot Richness (Deep Dive)

Events with `ai_suggestion_snapshot` populated are scored for documentation quality:

| Criterion | Score |
|-----------|-------|
| Has any snapshot at all | +0.3 |
| Also has `human_delta_summary` | +0.3 |
| Snapshot contains content keys (content, text, suggestion, draft, ai_output, ai_top_pick) | +0.2 |
| Decision weight is DEFINING | +0.2 |

The final richness score combines coverage (what proportion of all events have rich snapshots) × average per-event richness. Maximum 1.0.

### Tier Classification

| Tier | Composite Score | Meaning |
|------|----------------|---------|
| `STRONG` | ≥ 0.7 | Creator actively directs the work, well-documented divergence |
| `MODERATE` | ≥ 0.4 | Balanced human-AI collaboration |
| `LIGHT` | ≥ 0.2 | Creator leans heavily on AI, limited divergence |
| `INSUFFICIENT_DATA` | < 10 events | Not enough decisions to score |

### Summary Generation

Human-readable, factual and warm, never evaluative. Uses specific numbers: "This project involved 47 consequential creative decisions. You diverged from AI suggestions 63% of the time and contributed 2,400 words of original content." Adjusts language based on tier and component values.

---

## Layer 6: API Surface

**File:** `backend/app/api/v1/creator_profile.py`

### REST Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/creator/profile` | GET | Retrieve current profile. Returns 404 if < 20 events. |
| `/creator/profile/compute` | POST | Force full recomputation. Rate limited: 1/hour per tenant. |
| `/creator/provenance/{project_id}` | GET | Provenance score for a specific project. Returns 422 if insufficient data. |
| `/creator/provenance` | GET | Aggregate provenance across all projects. |
| `/creator/mirror` | GET | Mirror reflection as JSON with `reflection` (prose), `profile_ready` (bool), `growth_edges` (list), `ancestry_context`. |

### Telegram Handlers

| Command | Handler | Purpose |
|---------|---------|---------|
| `/mirror` | MirrorHandler | Generates and sends creative portrait as a Telegram message |
| `/provenance` | ProvenanceHandler | Sends ASCII provenance chart |

### Frontend Components (FE-1)

| Component | File | Purpose |
|-----------|------|---------|
| CreatorProfileCard | `frontend/src/components/dashboard/CreatorProfileCard.tsx` | Full profile display |
| ProvenanceScoreCard | `frontend/src/components/dashboard/ProvenanceScoreCard.tsx` | Provenance visualization |
| MirrorReflection | `frontend/src/components/dashboard/MirrorReflection.tsx` | Identity statement display |
| GrowthEdgesList | `frontend/src/components/dashboard/GrowthEdgesList.tsx` | Growth edge cards |

---

## Integration Points

### Where CDE Feeds Into Other Systems

**Alter Ego / Prompt Assembly:** The `persona_prompt` field on CreatorProfile (~200 chars) gets injected into the Alter Ego's system prompt via PromptAssembler. This is how the Alter Ego knows to be concise with a directive creator or exploratory with a collaborative one.

**Engagement Gate (CDE-4):** The onboarding engagement gate includes CDE signals: `decisions_made`, `original_words_added`, `ai_suggestions_modified`, `ai_suggestions_rejected`. Weight rebalancing: 30% active (CDE signals) + 70% passive (scroll, time, views). This means creators who actively participate in decisions pass the engagement gate faster.

**Graduation Service:** When a project graduates (completes), GraduationService triggers `CreatorProfileService.compute_incremental()`. The project's decisions become part of the cross-project profile.

**Weekly Scheduler:** A cron job triggers profile recomputation for all active tenants. This catches events that accumulated between project graduations.

**Taxonomy Graduation:** The `SKILL_PROMOTED_TO_TALENT`, `HABIT_PRACTICED`, `TALENT_PUBLISHED`, and `TALENT_INSTALLED` decision types feed back into the Talent Taxonomy's progression tracking. CDE is how the system knows a Skill has been used enough to warrant promotion to Talent.

---

## How to Add a New Decision Type

1. Add the value to the `DecisionType` enum in `backend/app/models/creative_decision.py`. No migration needed — the column is a String, not a PostgreSQL enum.

2. Choose the appropriate `DecisionWeight` (ROUTINE, SUBSTANTIVE, or DEFINING).

3. In the emitting service, create a `DecisionEmitter` and call `emit()` with a rich context dict. The context should include everything a future analyst would need to understand the decision — options available, what was chosen, what the AI recommended, why the human diverged.

4. If the new type should affect growth edge detection, update the relevant detector in `_detect_growth_edges()` or add a new detector (keeping the max at 5).

5. If the new type should affect provenance scoring, update the relevant component computation in `ProvenanceScoreService._compute_score()`.

6. Write tests that verify the event is emitted with the expected context shape.

---

## Proposed Extensions (SPEC-1/2/3)

Three specifications have been proposed to extend CDE. These are not yet implemented.

**SPEC-1: Intent Clarity (6th Growth Edge)**
Adds a new growth edge that detects whether a creator's instructions to the AI are becoming more precise over time. Measures spec quality — the distance between what the creator asked for and what they accepted. A creator whose specs are vague (and who accepts whatever comes back) gets a different growth edge than one whose specs are precise (and who rejects anything that doesn't match).

**SPEC-2: Adaptive Autonomy Profiles**
Computes an autonomy profile from CDE data: Scaffolded (the system guides heavily), Collaborative (balanced), or Autonomous (the creator leads). This profile controls how the Alter Ego behaves — how much it explains, how often it asks for confirmation, how proactively it suggests. The autonomy profile operates WITHIN the subscription tier, not across tiers. Tier defines the feature ceiling; autonomy defines the interaction style within that ceiling.

**SPEC-3: Spec Quality Scoring**
Infrastructure for measuring how well a creator's instructions predict their acceptance of results. Feeds SPEC-1's growth edge and SPEC-2's autonomy calculation.

---

## Testing

CDE tests are organized by component:

| Test File | Tests | Coverage |
|-----------|-------|----------|
| `test_creative_decision.py` | Event model, DecisionType enum, context serialization | ~20 |
| `test_decision_emitter.py` | Emission within transaction, persona trigger, fire-and-forget | ~15 |
| `test_creator_profile_service.py` | Full compute, incremental, each metric category, growth edges | ~40 |
| `test_provenance_score_service.py` | Each component, tiers, summary generation, snapshot richness | ~30 |
| `test_creator_profile_api.py` | REST endpoints, rate limiting, 404 on insufficient data | ~15 |
| `test_mirror_handler.py` | Telegram mirror output | ~10 |
| `test_engagement_tracker.py` | CDE-4 signal integration | ~9 |

All tests use MockLLMClient. Identity statement generation with a real LLM client has not been tested in CI.

---

## File Inventory

| File | LOC | Purpose |
|------|-----|---------|
| `backend/app/models/creative_decision.py` | ~142 | Event model + DecisionType + DecisionWeight enums |
| `backend/app/services/decision_emitter.py` | ~93 | Transactional event emission |
| `backend/app/models/creator_profile.py` | ~146 | Profile model (one per tenant) |
| `backend/app/schemas/creator_profile.py` | ~106 | Pydantic schemas for API responses |
| `backend/app/services/creator_profile_service.py` | ~1,005 | Profile computation, growth edges, voice trajectory, identity |
| `backend/app/services/provenance_score_service.py` | ~398 | Provenance scoring (6 components) |
| `backend/app/api/v1/creator_profile.py` | ~221 | REST endpoints |
| `backend/app/services/telegram_handlers/mirror_handler.py` | ~80 | Telegram /mirror command |
| `frontend/src/components/dashboard/CreatorProfileCard.tsx` | ~150 | Profile display component |
| `frontend/src/components/dashboard/ProvenanceScoreCard.tsx` | ~120 | Provenance visualization |
| `frontend/src/components/dashboard/MirrorReflection.tsx` | ~80 | Mirror display |
| `frontend/src/components/dashboard/GrowthEdgesList.tsx` | ~100 | Growth edge cards |
| `frontend/src/components/dashboard/EngagementTracker.tsx` | ~150 | CDE-4 engagement signals |

---

## Design Principles (Carry Forward)

1. **Portrait, not analytics.** Every surface that displays CDE data should translate numbers into human meaning. "0.68 divergence" → "you chose differently two-thirds of the time."

2. **Events are sacred.** Never update or delete CreativeDecisionEvents. They're historical facts. The profile is a derived view that can always be recomputed.

3. **Rich context or nothing.** A decision event with `{"decision": "accepted"}` in the context field is nearly worthless. The context should contain everything needed to understand the decision later: what options existed, what was chosen, what the AI recommended, and (if available) why the human diverged.

4. **Warm, not evaluative.** Growth edges are framed as invitations ("try this"), not criticisms ("you're doing this wrong"). Identity statements are specific and warm. Provenance tiers are descriptive ("LIGHT means you leaned on AI"), not judgmental ("LIGHT means low quality").

5. **The creator chooses.** A low provenance score is not bad. A high AI reliance is not wrong. CDE provides the mirror; the creator decides what to do with the reflection.

---

*This document describes CDE as implemented through CDE-4 (February 2026). SPEC-1/2/3 are proposed extensions. For the original design rationale, see `docs/plans/2026-02-14-creator-development-engine.md`.*
