# Backend Services Ready for Skill/Tool Conversion

> **Purpose:** Services that exist in the backend with real logic but lack user-facing wrappers (Skill registration, Telegram command, or dedicated UI surface). These are the lowest-hanging fruit for launch — the code is written, it just needs to be exposed.
> **Updated:** 2026-02-26
> **Rule:** When a service gets wrapped as a skill/tool, move it to [skills-and-tools-built.md](skills-and-tools-built.md).

---

## How to Read This Document

Each entry has:
- **Service:** The backend file that already exists
- **What it does:** Plain English
- **What's needed:** The wrapper work to make it user-facing
- **Effort:** Estimated LOC for the wrapper (not the service — that's built)
- **Priority:** Launch (must have), Month 1 (early adopters ask), or Later

---

## HIGH PRIORITY — Launch Readiness

### 1. Batch Crystallization
- **Service:** `crystallize_batch_service.py` (169 LOC)
- **What it does:** Processes multiple sources at once instead of one-by-one. "Crystallize my entire Obsidian vault."
- **What's needed:** Skill wrapper (~50 LOC), Telegram `/crystallize-all` command (~30 LOC), progress callback
- **Effort:** ~80 LOC
- **Priority:** Launch — power users hit the one-at-a-time wall immediately
- **Tier:** All

### 2. Contradiction Detection
- **Service:** `contradiction_detector.py` (exists, SPEC-002 complete with code)
- **What it does:** Detects semantic contradictions across Codex chunks. "Your character was born in 1985 here but 1987 there."
- **What's needed:** Skill wrapper, REST endpoint (`GET /contradictions`), Telegram notification on detection
- **Effort:** ~100 LOC
- **Priority:** Launch — knowledge governance is incomplete without this
- **Tier:** All (it's a Governor-level reflex)

### 3. Morning/Health Briefing
- **Service:** `health_briefing_service.py` (9 functions, fully implemented)
- **What it does:** Generates health and activity summaries from indexed life data
- **What's needed:** Skill wrapper as composable morning briefing, Telegram `/briefing` command. (Note: BRIEF-1 dispatched to Codex tonight may cover this)
- **Effort:** ~100 LOC
- **Priority:** Launch — every tier needs a lightweight daily touchpoint
- **Tier:** All

### 4. Export/Manuscript Assembly
- **Service:** `content_assembly_service.py` (exists), `gallery_export_service.py` (exists)
- **What it does:** Assembles scenes into ordered content, exports gallery items
- **What's needed:** Dedicated manuscript export skill (clean Markdown + EPUB). (Note: EXPORT-1 dispatched to Codex tonight)
- **Effort:** ~200 LOC
- **Priority:** Launch — writers must get work OUT of the system
- **Tier:** All

### 5. Conversation Crystallizer (already a skill, needs broader exposure)
- **Service:** `conversation_crystallizer_skill.py` (exists)
- **What it does:** Extracts decisions, patterns, and preferences from conversation transcripts into Codex
- **What's needed:** Better Telegram UX (`/crystallize` already exists), Web UI integration, auto-trigger after long conversations
- **Effort:** ~60 LOC (UI wiring)
- **Priority:** Launch — this is the Alter Ego's learning mechanism
- **Tier:** All

---

## MEDIUM PRIORITY — Month 1

### 6. Creator Profile / Mirror
- **Service:** `creator_profile_service.py` + `provenance_score_service.py` (CDE-2, fully built)
- **What it does:** Computes growth edges, provenance scores, and development profile
- **What's needed:** Already has REST + Telegram. Needs Web dashboard panel (FE-4 was deferred), scheduled auto-compute
- **Effort:** ~80 LOC (FE wiring)
- **Priority:** Month 1 — differentiator but not blocking
- **Tier:** Making+ (metacognition is a paid feature)

### 7. Habit Detection & Execution
- **Service:** `habit_executor.py` + `habit_pattern_detector.py` + `habit_personalizer.py`
- **What it does:** Detects behavioral patterns, creates habits, executes them on schedule
- **What's needed:** Skill wrapper, Telegram `/habits` command, cron registration
- **Effort:** ~120 LOC
- **Priority:** Month 1 — "the system learns your rhythms"
- **Tier:** Making+ (habit formation is basal ganglia = Tier 1)

### 8. Signal Aggregation Dashboard
- **Service:** `signal_aggregator_service.py` + 5 signal sources (health, creator, cost, synthetic, X)
- **What it does:** Aggregates cross-domain signals into a unified view
- **What's needed:** Skill wrapper, REST endpoint for dashboard, Telegram summary command
- **Effort:** ~100 LOC
- **Priority:** Month 1 — cross-domain intelligence is the paid-tier differentiator
- **Tier:** Meaning+ (limbic system = emotional/signal correlation)

### 9. Topic Queue / Content Calendar (enhanced)
- **Service:** `topic_queue_service.py` (exists, in-memory MVP)
- **What it does:** Suggest topics with time-decay priority, mark covered, detect duplicates
- **What's needed:** Persistent storage (PostgreSQL migration), richer Telegram commands, Web panel
- **Effort:** ~150 LOC (migration + UI)
- **Priority:** Month 1 — content creators need planning tools
- **Tier:** All (basic), Making+ (scheduled suggestions)

### 10. Cost Dashboard
- **Service:** `cost_dashboard_service.py` (exists)
- **What it does:** Daily cost trends, action breakdown, anomaly detection
- **What's needed:** Web dashboard component, Telegram `/costs` command (may already exist)
- **Effort:** ~80 LOC (FE wiring)
- **Priority:** Month 1 — transparency builds trust
- **Tier:** All

### 11. Text Network Visualization
- **Service:** `text_network_service.py` (614 LOC, Louvain community detection)
- **What it does:** Builds knowledge graph from text, detects communities, finds structural gaps
- **What's needed:** Web visualization component (force-directed graph), dedicated skill wrapper
- **Effort:** ~200 LOC (heavy on frontend)
- **Priority:** Month 1 — the "wow moment" for knowledge workers
- **Tier:** All (the wow moment should be free)

### 12. Intelligence Summary
- **Service:** `intelligence_summary_service.py` + `intelligence_prompts.py`
- **What it does:** Generates cross-source intelligence briefs (what patterns are emerging)
- **What's needed:** Skill wrapper, scheduled weekly generation, Telegram delivery
- **Effort:** ~100 LOC
- **Priority:** Month 1 — "the system sees patterns I don't"
- **Tier:** Meaning+ (this is limbic-level insight)

---

## LOWER PRIORITY — Quarter 1

### 13. Genealogy Abstraction Layer
- **Service:** `genealogy_abstraction_service.py` + `genealogy_crystallize_service.py`
- **What it does:** Abstract family tree analysis — identity patterns, migration paths, naming patterns
- **What's needed:** Skill wrapper, dedicated `/family` Telegram command set, Web family tree viewer
- **Effort:** ~150 LOC
- **Priority:** Q1 — niche but deep engagement
- **Tier:** All

### 14. Voice Resolver (multi-character)
- **Service:** `voice_resolver.py`
- **What it does:** Selects the right voice profile for each character in multi-POV work
- **What's needed:** Better UI for managing multiple voice profiles, export as Voice Pack
- **Effort:** ~100 LOC
- **Priority:** Q1 — advanced feature for novelists/screenwriters
- **Tier:** Making+

### 15. Moltbook Social Engagement
- **Service:** `moltbook_service.py` + `moltbook_sensor.py` + 11-service engagement submodule
- **What it does:** Social media presence management, risk scoring, content categorization, draft generation
- **What's needed:** Already fairly exposed. Needs Web dashboard panel.
- **Effort:** ~80 LOC (FE wiring)
- **Priority:** Q1
- **Tier:** Meaning+

### 16. NSL Bridge (Narrative Structure Lab)
- **Service:** `nsl_bridge_service.py`
- **What it does:** Loads parsed TV script data, search beats by genre/act, get dialogue exemplars
- **What's needed:** Skill wrapper for "learn from TV shows" use case
- **Effort:** ~80 LOC
- **Priority:** Q1 — niche but beloved by screenwriters
- **Tier:** All

### 17. Birthday Pipeline Orchestrator
- **Service:** `birthday_pipeline_orchestrator.py`
- **What it does:** End-to-end birthday video creation from photos + stories
- **What's needed:** Web wizard UI, skill wrapper, progress tracking
- **Effort:** ~200 LOC
- **Priority:** Q1 — high-emotion use case
- **Tier:** All

### 18. Self-Audit / Structural Metrics
- **Service:** `self_audit_service.py` + `structural_metrics_service.py`
- **What it does:** Code quality and structural analysis of the codebase itself
- **What's needed:** This is more Director-facing than user-facing. Could be a Dev Talent.
- **Effort:** ~60 LOC
- **Priority:** Q1 (Vibe Coder tier)
- **Tier:** Mastery

### 19. Humanizer
- **Service:** `humanizer_service.py`
- **What it does:** Makes AI-generated text sound more human
- **What's needed:** Skill wrapper, simple "humanize this" command
- **Effort:** ~50 LOC
- **Priority:** Q1 — utility skill
- **Tier:** All

### 20. Cover Generator
- **Service:** `cover_generator.py` + `image_generation_service.py`
- **What it does:** Generates book/comic covers using multi-provider image generation
- **What's needed:** Dedicated skill wrapper, Web preview UI
- **Effort:** ~100 LOC
- **Priority:** Q1 — visual output is high-engagement
- **Tier:** All

---

## Infrastructure Services (NOT Skill Candidates)

These are correctly infrastructure-only. They power other services but should never be user-facing skills.

| Service | Why Not a Skill |
|---------|----------------|
| `llm_client.py`, `llm_service.py`, `model_catalog.py` | Model routing infrastructure |
| `provider_registry.py`, `provider_router.py`, `provider_health.py` | Provider management |
| `prompt_assembler.py` | System prompt construction |
| `billing_service.py`, `paddle_billing.py`, `credit_ledger_service.py` | Payment infrastructure |
| `budget_orchestrator.py`, `surface_access_service.py` | Tier enforcement |
| `oauth_service.py`, `tenant_provision_service.py` | Auth infrastructure |
| `skill_executor.py`, `skill_validator.py`, `skill_sandbox.py` | Skill runtime |
| `atomic_write_manager.py`, `outbox_relay.py` | Transaction infrastructure |
| `pipeline_context_builder.py`, `pipeline_worker.py` | Pipeline runtime |
| `observation_service.py`, `health_check_service.py` | System monitoring |
| `scheduler_service.py`, `notification_delivery_service.py` | Background job infrastructure |
| `workspace_manager.py`, `sandbox_manager.py` | Execution environment |
| `key_provisioning_service.py`, `permission_validator.py` | Security infrastructure |
| `agent_router.py`, `agentic_core.py`, `agentic_loop.py` | Agent runtime |

---

## Summary

| Priority | Count | Est. Total LOC |
|----------|-------|---------------|
| **Launch (must have)** | 5 services | ~540 LOC |
| **Month 1** | 7 services | ~810 LOC |
| **Quarter 1** | 8 services | ~820 LOC |
| **Total conversion work** | 20 services | ~2,170 LOC |

The launch-critical items (batch crystallize, contradiction detection, morning briefing, manuscript export, conversation crystallizer UX) are all under 600 LOC combined. Two of these (BRIEF-1, EXPORT-1) are already dispatched to Codex tonight.
