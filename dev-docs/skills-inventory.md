# Life-OS Skills Inventory

> Complete inventory of implemented and specced skills for Life-OS / Agent-C.
> **Last Updated:** February 26, 2026
> **Update Rule:** When a new skill is implemented or specced, add it here.
> **Companion docs:** [capabilities-inventory.md](capabilities-inventory.md) (paid app layer) | `docs/plans/2026-02-22-engine-taxonomy-relationship-map.md` (engine/métier architecture)

Skills serve all engine categories: Creative Engines (Writer, Filmmaker, Comic Artist), Vibe Coder Engine, Operator Engines (Marketing, Business, Clinical Research, Education, Legal, Finance, Trade/Service, and any domain where expertise can be codified), Knowledge Engines (Second Brain, Research), and Life Data Engines (Body, Perception, Consumption). The same skill behaves differently depending on the active engine — Brainstorm generates story premises in the Writer's Engine but visual concepts in the Filmmaker's Engine. See the engine taxonomy map for the full architecture.

## Terminology

| Term | Meaning | Where It Appears |
|------|---------|-----------------|
| **Talent** | Agent-C's native, validated, governed capability | UI, marketing, docs |
| **Skill** | External SOP from other platforms (Claude, OpenClaw, MCP, OpenAI) — unvetted | Backend code, imports |
| **Tool** | Raw capability (API call, file operation, database query) | Backend internals |

Backend code says "skill" everywhere. User-facing language says "talent." Same thing, different audience.

## Implemented Skills

### Onboarding & Discovery

| Skill | Location | Description |
|-------|----------|-------------|
| **Format Picker** | `services/skills/format_picker_skill.py` | Recommends creative format (novel, memoir, TikTok, etc.) based on genre, audience, length |
| **Engagement Analysis** | `services/skills/engagement_analysis_skill.py` | Evaluates creator engagement signals against format-specific gates. Returns pass/fail |
| **Content Calendar** | `services/skills/content_calendar_skill.py` | Topic suggestions with time-decay priority via TopicQueueService |

### Knowledge Management

| Skill | Location | Description |
|-------|----------|-------------|
| **Conversation Crystallizer (SK-02)** | `services/conversation_crystallizer_skill.py` | Extracts decisions, patterns, preferences from conversations → Codex. Spec: `docs/plans/2026-02-17-conversation-crystallizer-skill.md` |
| **Sovereign Memo (SK-03)** | `services/skills/sovereign_memo_skill.py` | Three-ring URL analysis: tenant URLs, Director memos, interactive triage. Analyzes against project context + architecture |
| **URL Intelligence** | `services/url_intelligence_skill.py` | General variant of Sovereign Memo. Routes URL content to conversation, memo, or task extraction |
| **Inbox Router** | `services/inbox_router_skill.py` | Routes incoming files/emails to triage categories (part of SK-01 framework) |

### Health & Identity Data

| Skill | Location | Description |
|-------|----------|-------------|
| **Genetic Health Analyzer** | `services/skills/genetic_health_skill.py` | 23andMe raw data against 79 curated SNPs + ClinVar (341k variants) + PharmGKB. Modes: full, lifestyle, disease, protocol. `forever_free` governance flag. Spec: `docs/plans/2026-02-25-genetic-health-skill-spec.md` |
| **FamilySearch Search** | `skills/familysearch_search/SKILL.md` | Read-only genealogy lookup for memoir fact-checking. Tools: search_person, get_person_brief, get_pedigree_preview. No auth required (platform app key) |

### Operations & Intelligence

| Skill | Location | Description |
|-------|----------|-------------|
| **State of the Union** | `skills/state_of_the_union/SKILL.md` | End-of-session narrative report. Context-adaptive (development, creative, business, research, personal). Includes crystallization to Codex. Schedule: 8pm daily |
| **Task Manager** | `skills/task_manager/SKILL.md` | Tracks and prioritizes project tasks. Actions: report, overdue, summary, next. Schedule: 9am weekdays |
| **Daily Briefing** | `skills/daily_briefing/SKILL.md` | Morning intelligence update — overdue tasks, schedule, sprint goals. Schedule: 8am daily |
| **CES Director** | `skills/_ces_director/SKILL.md` | Chief of Staff for CES project flow. Workflows: /status, /dispatch, /gap, /merge, /brainstorm. Internal only |
| **Skill Scout** | `skills/skill-scout/SKILL.md` | Discovers MCP servers, AI tools, competitor capabilities. Evaluates maturity, security, relevance. Output: IMPORT/BUILD/WATCH/SKIP recommendations |

## Specced (Design Complete, Not Yet Built)

| Skill | Spec | Description |
|-------|------|-------------|
| **Nightly Council** | `docs/plans/2026-02-26-nightly-council-skill-template.md` | Society of Thought applied to operations. Three seed councils: Platform Health, Creative Quality, Growth. Scheduled perspective-based audits |
| **Skill Portability & ACV** | `docs/plans/2026-02-17-skill-portability-acv-spec.md` | Automatic Classified Validation for importing external skills. Risk tiers: prompt-only, tool-using, executable |
| **Plugin-to-Skill Conversion** | `docs/plans/2026-02-19-plugin-to-skill-conversion-pipeline.md` | Converts Claude plugins to Agent-C talents. Ingestion → Analysis → Validation → Governance → Approval |

## Infrastructure (Internal, Not User-Facing)

These services make the skill system work. Developers building new skills don't need to touch these unless extending the framework.

| Service | Location | Purpose |
|---------|----------|---------|
| **SkillExecutor** | `services/skill_executor.py` | Routes skill execution, async handling, error recovery |
| **SkillValidator** | `services/skill_validator.py` | Validates skill metadata and contract compliance |
| **SkillSandbox** | `services/security/skill_sandbox.py` | Sandboxes code execution for Tier 3 (executable) skills |
| **SkillRiskClassifier** | `services/skill_risk_classifier.py` | Classifies malicious patterns, permission analysis |
| **SkillGovernorBridge** | `services/skill_governor_bridge.py` | Wires skills to Narrative Governor N1-N5 |
| **SkillImportAdapter** | `services/skill_import_adapter.py` | Imports external skill manifests |
| **SkillExportAdapter** | `services/skill_export_adapter.py` | Exports Agent-C talents to external platforms |
| **SkillDataService** | `services/skill_data_service.py` | GCS shared reference data + local/memory cache for skills with large datasets |
| **SkillFacadeMapper** | `services/skill_facade_mapper.py` | Maps component facades (ECS transition pattern) |
| **Skill Registry (SK-01)** | `services/skills/registry.py` | Code-registered skill lookup. Foundation for all skills |

## Knowledge Connectors (Second Brain / PKMS)

Fully implemented two-phase connectors: Index (metadata, fast) → Selective Crystallization (content on demand). All exposed via `/api/v1/knowledge.py`.

| Connector | Location | Description |
|-----------|----------|-------------|
| **ObsidianIngestService** | `services/obsidian_ingest_service.py` (~415 LOC) | Reads vault filesystem directly (no export). Parses YAML frontmatter, extracts `[[wiki-links]]` as graph edges. Two-phase: ObsidianIndexService + ObsidianCrystallizeService |
| **RoamIngestService** | `services/roam_ingest_service.py` (~474 LOC) | Accepts JSON upload. Block-level crystallization with parent lineage breadcrumbs. Sanitizes Roam artifacts (kanban, embeds, block refs). Built-in LogseqNormalizer (~30 LOC) |
| **ReadwiseIngestService** | `services/readwise_ingest_service.py` (~465 LOC) | OAuth API integration. Contextual Donut pattern: highlight (core) + surrounding text (bread) + user annotation (glaze). Cursor-based pagination |
| **TextNetworkService** | `services/text_network_service.py` (~614 LOC) | NetworkX graph from SourceIndex metadata. Louvain community detection, degree centrality, bridge insight generation. Replaces InfraNodus (built, not bought) |

**Key design decisions:**
- **Obsidian reads the filesystem directly** — no export step, wiki-links give graph structure for free
- **Roam/Logseq accept JSON upload** — user controls what gets sent, avoids API auth complexity
- **Readwise uses OAuth API** — no export needed
- **Blocks are the atomic unit** for Roam/Logseq (not pages) — each block becomes one CodexChunk
- **Metadata-first analysis** — graph built from explicit links/tags, not word co-occurrence
- **Content hashing** — SHA-256 deduplication across re-runs

## Life Data Skills (Specced — from Life-Data Two-Tier Taxonomy)

These skills implement the two-tier life data taxonomy: Level 1 (Body — physical existence, routines, location) and Level 2 (Mind — creative consumption, reading, entertainment, social patterns). Spec: `docs/sovereign-memos/2026-02-23-life-data-two-tier-skill-taxonomy.md`.

### Shared Infrastructure

| Skill | Spec Task | Description |
|-------|-----------|-------------|
| **CSV Import Service** | LD-1 (~200 LOC) | Generic CSV import: file upload, format detection (Netflix/Kindle/Audible auto-detect), parsing, SourceIndex creation. Handles platforms with no API (manual export → upload). |

### Level 2 — Mind (Creative Consumption & Intellectual)

| Skill | Spec Task | Description |
|-------|-----------|-------------|
| **Netflix Viewing History** | LD-2 (~80 LOC) | Netflix CSV parser. Series/season/episode extraction, date parsing, binge pattern detection (3+ episodes same series in one day). Guided upload instructions. |
| **Kindle Reading History** | LD-3 (~100 LOC) | Amazon data export parser. Two CSV types: DocumentMetadata (library) + ReadingSession (timestamps, pages, progress). Calculates reading velocity, completion rate, genre shifts. |
| **Audible Listening History** | LD-4 (~80 LOC) | Amazon data export parser. Listening sessions from Audible.Listening.zip. Completion rate, listening speed, preferred genres, time-of-day patterns. |
| **Social History Aggregator** | LD-6 (~100 LOC) | Parser for social media archive exports: Twitter/X (tweets.js), LinkedIn (Connections.csv), Facebook (your_posts.json). Topic extraction, engagement patterns, network analysis. |
| **Reading & Research Tracker** | LD-7 (~120 LOC) | Unified reading profile: aggregates Kindle + Audible + future sources (Goodreads, Pocket). Cross-medium analysis: velocity trends, genre shifts, completion patterns, research clustering. |

### Level 1 — Body (Physical Existence & Routines)

| Skill | Spec Task | Description |
|-------|-----------|-------------|
| **GPS / Location History** | LD-5 (~120 LOC) | Google Takeout location JSON parser. Clusters locations into "places," detects travel periods, time-in-city aggregation. Correlates with creative output for location-creativity insights. |

### Cross-Tier Correlation

| Skill | Spec Task | Description |
|-------|-----------|-------------|
| **Consumption → Production Correlation** | LD-8 (~150 LOC) | Highest-leverage component. Cross-references Level 1 (body) and Level 2 (mind) data: reading vs writing patterns, sleep vs creative quality, location vs output, entertainment consumption vs project starts. Generates life briefings synthesizing all available data. |

**Key design decisions:**
- Netflix/Kindle/Audible have NO APIs — all require CSV manual export, introducing the import-from-CSV connector pattern (vs pull-from-API)
- The CSV Import Service (LD-1) handles format detection and validation for all CSV-based connectors
- Guided upload instructions included per platform (step-by-step for each export process)
- Content hashing for deduplication on re-upload
- Total: ~950 LOC, ~88 tests across 8 tasks. Critical path: LD-1 → (LD-2, LD-3, LD-4 parallel) → LD-7 → LD-8

## Not Yet Wrapped as Skills

These core creative services are implemented as backend services and work through agent routing + REST, but aren't registered in the skill framework. Wrapping them would make them composable in recipes.

| Service | What It Does | Why It Matters |
|---------|-------------|----------------|
| **BrainstormService** | Generates creative directions with context injection | Recipe: brainstorm → voice cal → generation |
| **VoiceCalibrationService** | Tournament-based voice matching | Recipe: calibrate → lock → generate in voice |
| **SceneWriterService** | Generates scenes via Director Mode | Recipe: scaffold → write → governor audit |
| **GovernorService** | N1-N5 narrative audit | Recipe: generate → audit → ORA remediation |
| **VideoDirectorService** | Orchestrates video generation pipeline | Recipe: script → shot plan → render → tournament |
| **CreativeRefinementOrchestrator** | Generic generate → critique → revise loop | Recipe: any generator + any critic |

## Skill File Conventions

**SKILL.md skills** (agentic, prompt-based): Live in `backend/app/skills/<skill-name>/SKILL.md`. These define tool descriptions, safety rules, and behavioral instructions. The LLM reads these at invocation time.

**Code-registered skills** (Python classes): Live in `backend/app/services/skills/`. These implement `execute()` methods with structured inputs/outputs. Registered in `registry.py`.

**Adding a new skill:** Follow the InboxRouterSkill pattern in `services/skills/` for code skills, or the FamilySearch pattern in `skills/` for SKILL.md-based agentic skills. Both types must be registered to be discoverable.
