# Skills, Tools & Connectors — Already Built

> **Purpose:** Complete inventory of user-facing capabilities already wired into the platform.
> **Updated:** 2026-02-26
> **Rule:** When a new skill/tool ships, add it here with the service file, tier gate, and surface.

---

## How to Read This Document

Everything listed here is **implemented and callable** — either through the REST API, Telegram commands, or the Skill Executor. These are not backend-only services; they have user-facing entry points.

**Categories:**
- **Skills** — Registered in the Skill Executor, callable via `/skills/{slug}/run`
- **Tools** — Available to the Agent (architect/user agent) as callable functions
- **Connectors** — Source data integrations (OAuth or file upload → Index → Crystallize pipeline)
- **Sinks** — Output destinations (publish/export to external platforms)
- **Cron Jobs** — Scheduled background processes that run automatically

---

## 1. Formal Skills (Skill Executor Registry)

These are registered in the `skill` table and executed by `SkillExecutor`. Each has a SKILL.md or handler.py.

| Skill | What It Does | Service File | Tier | Surface |
|-------|-------------|-------------|------|---------|
| **Inbox Router** | Routes incoming content to the right engine/pipeline based on content type | `inbox_router_skill.py` | All | API, Telegram |
| **URL Intelligence** | Analyzes a URL — extracts content, classifies it, suggests how to use it | `url_intelligence_skill.py` | All | API, Telegram |
| **Content Calendar** | Suggests topics with time-decay priority, tracks what's been covered | `content_calendar_skill.py` | All | API |
| **Format Picker** | Recommends output format based on source material and creator profile | `format_picker_skill.py` | All | API, Web |
| **Engagement Analysis** | Scores engagement signals (time, scroll, interactions) for conversion gating | `engagement_analysis_skill.py` | All | API |
| **Sovereign Memo** | Files knowledge artifacts to the Codex with triage classification | `sovereign_memo_skill.py` | All | API, Telegram |
| **Genetic Health** | Analyzes 23andMe raw data — 79 SNPs, ClinVar cross-reference, governance flags | `genetic_health_skill.py` | All | API |
| **FamilySearch Search** | Searches FamilySearch genealogy database by name/dates/location | `familysearch_search_service.py` | All | API, Telegram |

---

## 2. Agent Tools (Available to Alter Ego / Director Agent)

These are callable functions wired into the `architect_agent.py` or `user_agent.py` tool lists.

| Tool | What It Does | Service File | Available To |
|------|-------------|-------------|-------------|
| **YouTube Transcript** | Fetches and returns transcript for a YouTube video | `youtube_transcript_tool.py` | Both agents |
| **Memory Search** | Searches conversation memory and Codex for relevant context | `memory_tools.py` | Both agents |
| **Subagent Dispatch** | Spawns specialized sub-agents for complex tasks | `subagent_tools.py` | Director agent |
| **Task Management** | Create, update, complete tasks (supports batch IDs) | `task_service.py` | Director agent |
| **Pipeline Context** | Assembles full pipeline state for a tenant — stage detection, progress | `pipeline_context_builder.py` | Both agents |
| **Recipe Executor** | Runs a multi-step recipe (chain of skills) | `recipe_executor.py` | Both agents |
| **Chain Executor** | Runs a defined YAML chain of operations | `chain_executor_service.py` | Both agents |

---

## 3. Source Connectors (Sensory Organs)

Each connector follows the Index → Crystallize → Codex pattern. "Index" ingests raw data into `SourceIndex`. "Crystallize" extracts structured knowledge into `CodexChunk`.

### OAuth API Connectors (automatic sync)

| Connector | Brain Analog | Index Service | Crystallize Service | Status |
|-----------|-------------|--------------|-------------------|--------|
| **Gmail** | Language (Wernicke's) | `gmail_index_service.py` | `gmail_crystallize_service.py` | Full pipeline |
| **Google Drive** | Language (Wernicke's) | `drive_index_service.py` | `drive_crystallize_service.py` | Full pipeline |
| **YouTube (owned)** | Vision + Hearing | `youtube_source_service.py` | via Crystallization | Full pipeline |
| **YouTube (liked)** | Taste (consumption) | `youtube_liked_index_service.py` | `youtube_liked_crystallize_service.py` | Full pipeline |
| **YouTube (RSS)** | Social feed | `youtube_rss_service.py` | — | Index only |
| **Pinterest** | Vision (aesthetic) | `pinterest_board_index_service.py` | `pinterest_board_crystallize_service.py` | Full pipeline + aesthetic profiling |
| **FamilySearch** | Interoception (identity) | `familysearch_index_service.py` | `familysearch_crystallize_service.py` | Full pipeline (FS-0→FS-7) |
| **Fitbit** | Proprioception (body) | `fitbit_index_service.py` | via health pipeline | Full pipeline |
| **Oura** | Proprioception (body) | `oura_index_service.py` | via health pipeline | Full pipeline |
| **Medium** | Taste (reading) | `medium_index_service.py` | `medium_crystallize_service.py` | Full pipeline |
| **Medium (RSS)** | Taste (reading) | `medium_rss_index_service.py` | — | Index only |
| **Medium (Export)** | Taste (reading) | `medium_export_index_service.py` | — | Index only |
| **Readwise** | Language (highlights) | `readwise_index_service.py` | `readwise_crystallize_service.py` | Contextual Donut pattern |
| **Pocket** | Taste (saved intent) | `pocket_index_service.py` | `pocket_crystallize_service.py` | Full pipeline |
| **Twitter/X** | Social perception | `twitter_bookmark_index_service.py` | `twitter_bookmark_crystallize_service.py` | Bookmarks pipeline |
| **X Source** | Social perception | `x_source_service.py` | — | Source adapter |
| **Instagram** | Vision (social) | `instagram_index_service.py` | `instagram_crystallize_service.py` | Full pipeline |
| **Reddit** | Social (discussion) | `reddit_index_service.py` | `reddit_crystallize_service.py` | Full pipeline |
| **TikTok** | Vision + Taste | `tiktok_index_service.py` | `tiktok_crystallize_service.py` | Full pipeline |
| **Polymarket** | Prediction signals | `polymarket_source_service.py` | via brainstorm | 500-char context injection |
| **Google Calendar** | Memory (temporal) | `calendar_index_service.py` | — | Index (Codex pending) |

### File Upload / Import Connectors

| Connector | Brain Analog | Service | Status |
|-----------|-------------|---------|--------|
| **Obsidian Vault** | Language (structured) | `obsidian_ingest_service.py` (415 LOC) | Wiki-link graph extraction |
| **Roam Research** | Language (structured) | `roam_ingest_service.py` (474 LOC) | Block-level crystallization |
| **Logseq** | Language (structured) | Logseq normalizer (~30 LOC on Roam) | Adapter on Roam pipeline |
| **NotebookLM** | Language (synthesized) | `notebooklm_bridge_service.py` | Triplet extraction (Phase F2.0) |
| **23andMe** | Interoception (genetic) | `genetic_health_skill.py` | 79 SNPs + ClinVar |
| **GEDCOM** | Interoception (genealogy) | `gedcom_parser_service.py` | Family tree parsing |
| **CSV (generic)** | Various | `csv_import_service.py` | Auto-detect Netflix/Kindle/Audible/Goodreads |
| **Conversation Import** | Language (transcripts) | `conversation_import_service.py` | Claude/ChatGPT/Grok/Perplexity formats |
| **CLI Transcript Import** | Language (terminal) | CLI adapters (7 providers) | Gemini/Claude/Codex/Deepseek/Qwen |

**Connector count: 30+ implemented** (21 OAuth API + 9 file/import)

---

## 4. Output Sinks (Efferent Pathways)

These publish content to external platforms. All implement `BaseSink`.

| Sink | Destination | Format | Service File |
|------|-------------|--------|-------------|
| **Substack** | Substack newsletter | HTML/text | `substack_sink.py` + `substack_client.py` |
| **Newsletter** | Generic email newsletter | HTML | `newsletter_sink.py` + `newsletter_client.py` |
| **Medium** | Medium blog | Markdown → API | `medium_sink.py` |
| **YouTube** | YouTube upload | Video | `youtube_video_sink.py` + `youtube_upload_client.py` |
| **TikTok** | TikTok upload | Video | `tiktok_video_sink.py` + `tiktok_upload_client.py` |
| **Veo** | Google Veo AI | Video generation | `veo_video_sink.py` |
| **Kling** | Kling AI | Video generation | `kling_video_sink.py` |
| **Higgsfield** | Higgsfield | Video generation | `higgsfield_video_sink.py` |
| **Seedance** | Seedance | Video generation | `seedance_video_sink.py` |
| **Aggregator** | Multi-provider | Video routing | `aggregator_video_sink.py` |
| **Moltbook** | Moltbook social | Social media | `moltbook_sink.py` |
| **Remotion** | Remotion render | Video composition | `remotion_sink.py` |
| **Fountain** | Fountain format | Screenplay | `fountain_sink.py` |
| **KDP Ebook** | Amazon KDP | EPUB | `kdp_ebook_sink.py` |
| **ElevenLabs** | ElevenLabs TTS | Audiobook (M4B) | `elevenlabs_audiobook_sink.py` |
| **Telegram** | Telegram chat | Messages/media | `telegram_sink.py` |

**Sink count: 16 export destinations**

---

## 5. Scheduled Jobs (Cron / Background)

| Job | Schedule | What It Does | Service |
|-----|----------|-------------|---------|
| **Weekly Mirror** | Weekly | Generates CDE mirror reflection for creator | `weekly_mirror_job.py` |
| **Health Check** | Configurable | PostgreSQL + Neo4j + LLM provider health | `health_check_service.py` |
| **Weekly Summary** | Weekly | 7-day job log aggregation | `health_check_service.py` |
| **Proactive Notifications** | Configurable | Stuck creators, expiring subs, new signups | `proactive_notification_service.py` |
| **Standing Orders** | Per-order cron | Recurring reminders/tasks | `standing_orders_service.py` |

---

## 6. Governance Capabilities (Always Free)

These run at every tier — they're the "spinal reflexes" of the system.

| Capability | What It Does | Service |
|-----------|-------------|---------|
| **N1-N5 Audit** | Narrative violation detection (character contamination, continuity breaks, arc violations, voice drift, pacing failure) | `governor_service.py` |
| **Kill Switches** | Arc Wholeness, Character Integration, Setup-Payoff Distance checks | `governor_enforcer.py` |
| **ORA Loop** | Observe-Reason-Act iterative correction until scene passes governance | `creative_refinement_orchestrator.py` |
| **ClosedClaw Verification** | Security scanning of imported skills | `skill_validator.py` + `skill_sandbox.py` + `skill_risk_classifier.py` |
| **Video Safety** | Content safety checks on video output | `video_safety_governor.py` |

---

## 7. Creative Pipeline Services (User-Accessible via API)

These aren't formal "skills" but are fully exposed through REST endpoints and used by the Agent.

| Service | What It Does | API Endpoint | Tier |
|---------|-------------|-------------|------|
| **Brainstorm** | Context-aware idea generation (NotebookLM + Pinterest + Polymarket injection) | `POST /brainstorm/generate` | All |
| **Interview** | Multi-turn creator interview for learning | `POST /interview/start` | All |
| **Story Bible** | Extract characters, locations, themes from sources | `POST /story-bible/generate` | All |
| **Voice Calibration** | Tournament-based voice training (3 strategies × temp range) | `POST /voice-calibration/run` | All |
| **Scene Generation** | Director-mode scene writing with governance | `POST /scenes/generate` | All |
| **Bookends** | Ch1 + Final Chapter + 25-chapter outline | `POST /bookends/generate` | All |
| **Trailer** | 3-scene video trailer from bookends (engagement-gated) | `POST /trailer/generate` | Engagement gate |
| **Comic Scaffold** | Panel layout, visual style canon, comic issue structure | `POST /comic/scaffold` | All |
| **Video Draft** | Multi-variant video generation | `POST /video/draft` | All |
| **Video Tournament** | Tournament selection from video variants | `POST /video/tournament` | All |
| **Video Final** | Final render with selected variant | `POST /video/final` | All |
| **Quick Video** | Fast one-shot video from text | `POST /quick-video/generate` | All |
| **Crystallization** | Transform raw sources into Codex knowledge | `POST /crystallizer/crystallize` | All |
| **Graduation** | Promote project knowledge to sovereign Codex | `POST /graduation/graduate` | All |
| **CDE Mirror** | Creator development reflection | `GET /creator/mirror` | All |
| **CDE Profile** | Growth edges, provenance score | `GET /creator/profile` | All |
| **Federated Search** | Cross-source search across all indexed data | `GET /knowledge/search` | All |
| **Text Network** | Louvain community detection on knowledge clusters | `POST /knowledge/network` | All |
| **Gallery** | Visual portfolio of generated content | `GET /gallery` | All |

---

## Summary Counts

| Category | Count |
|----------|-------|
| Formal Skills (Skill Executor) | 8 |
| Agent Tools | 7 |
| Source Connectors (OAuth) | 21 |
| Source Connectors (File/Import) | 9 |
| Output Sinks | 16 |
| Scheduled Jobs | 5 |
| Governance Capabilities | 5 |
| Creative Pipeline Services (API) | 19+ |
| **Total user-facing capabilities** | **90+** |
