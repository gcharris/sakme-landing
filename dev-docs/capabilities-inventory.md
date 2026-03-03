# Agent-C Capabilities Inventory

> Complete inventory of Agent-C paid app layer capabilities, tier gating, and what differentiates the beautiful app from the free Life-OS engine.
> **Last Updated:** February 26, 2026
> **Update Rule:** When a new paid capability is implemented or specced, add it here.
> **Companion doc:** [skills-inventory.md](skills-inventory.md) (free engine layer)

## The Monetization Boundary

**Experience, not capability.** The engine is identical across Life-OS and Agent-C. Free users who can wire it themselves get everything. Paid users get the beautiful app that makes it effortless.

| Layer | Name | For | What They Get |
|-------|------|-----|---------------|
| **Free** | Life-OS (AGPL-3.0) | Developers | Full engine + CLI + Telegram/WhatsApp/Discord/MCP. Self-host or BYOK. |
| **Paid** | Agent-C (Beautiful App) | Creators | Capacitor native app, Alter Ego UI, visual panels, marketplace, managed hosting |

Life-OS gives you the engine. Agent-C gives you the cockpit.

## Subscription Tiers

> **Note:** Tier names are under active review. See `docs/brainstorming/2026-02-26-subscription-tier-brainstorm.md` for the full analysis. The names below are current code values pending Director decision.

| Tier | Current Code Name | Price | Surfaces | Models | Key Limits |
|------|------|-------|----------|--------|------------|
| **Free** | EXPLORER | $0 | Web only | BYOK Gemini key | Limited projects, BYOK only |
| **Paid** | CREATOR | $15-19/mo | Capacitor app (iOS + Android + Web) + Telegram | Hosted (included) | All engines, persistent Alter Ego, credits included |
| **Paid** | SHOWRUNNER | $39-49/mo | All surfaces + API + MCP | Hosted (priority) | Unlimited, marketplace publishing |

## The Engine Platform (Métiers)

Agent-C is NOT a narrative engine with extras. It is a **creative engine platform** serving displaced knowledge workers across unlimited domains. Each engine (user-facing: "métier") is a configuration of the shared foundation — different governance profile, workbench layout, source/sink affinity, and onboarding path.

The Agency Era Manifesto defines two primary paths out of displacement — **Creators** (people with stories, ideas, and worlds to build) and **Operators** (domain experts whose jobs evaporated but whose knowledge didn't). Most people walk both paths. The engine architecture serves both equally.

### The Two Paths: Creators and Operators

**Path A — Creators** have creative output: stories, films, comics, music, podcasts. They need infrastructure that remembers their voice, governs their quality, and grows with their craft.

**Path B — Operators** have domain expertise: clinical trial methodology, curriculum design, financial modeling, plumbing codes, legal compliance, social media strategy. They package that expertise into AI-powered Talents that others can use. An Operator doesn't need narrative governance — they need domain integrity governance.

**Most people walk both paths.** The journalist builds research Talents during the day and writes a memoir at night. The displaced consultant packages strategy frameworks as Operator Talents and produces a business podcast.

### Creative Engines (For Creators — MVP)

| Engine | Who It Serves | Governance Profile | Readiness |
|--------|-------------|-------------------|-----------|
| **Writer's** | Novelists, memoirists, newsletter writers, bloggers, screenwriters | N1-N5 Narrative | ~90% |
| **Filmmaker's** | Short-drama creators, video essayists, documentary makers | Visual Continuity | ~75% |
| **Comic Artist's** | Webtoon creators, graphic novelists, visual storytellers | Panel Consistency | ~70% |

Future creative engines: Music, Podcast, Game Design, Photography — any creative métier where governed AI output improves the craft. Community-created via YAML definitions.

### Vibe Coder Engine (For Builders — Own Category)

| Engine | Who It Serves | Governance Profile | Readiness |
|--------|-------------|-------------------|-----------|
| **Vibe Coder's** | Creator-developers, automation builders, experimenters, anyone who builds by describing not typing | Code Drift | ~30% |

Vibe Coders are their own category — not purely creative (they don't produce narrative), not purely operational (they build tools, not package expertise). They represent the Manifesto's thesis in action: one person with the right AI infrastructure can build what previously required a team. The Vibe Coder engine governs code quality, architecture memory, and test integrity.

### Operator Engines (For Domain Experts — The Manifesto's Other Path)

The Manifesto says: "A former pharmaceutical researcher builds a clinical trial protocol validator. A retired teacher builds a curriculum alignment checker. A laid-off financial analyst builds a portfolio rebalancing advisor." Operators span every domain where human expertise can be codified into governed AI workflows (Talents).

| Engine | Who It Serves | Governance Profile | Readiness |
|--------|-------------|-------------------|-----------|
| **Marketing** | Content marketers, social media managers, brand strategists | Brand Consistency | Planned |
| **Business** | Founder-operators, freelancers, consultants, project managers | Operational Integrity | Planned |
| **Social Media** | Social network managers, community builders, influencer strategists | Audience Consistency | Planned |

**The Operator category is explicitly unbounded.** The architecture supports engines for ANY domain where expertise can be packaged:

| Potential Engine | Who It Serves | Domain Knowledge |
|------------------|-------------|-----------------|
| Clinical Research | Pharma researchers, clinicians, trial coordinators | Protocol validation, regulatory compliance, literature review |
| Education | Teachers, curriculum designers, trainers, tutors | Scaffolding, alignment checking, assessment design |
| Legal | Lawyers, paralegals, compliance officers, contract specialists | Contract review, regulatory tracking, case research |
| Finance | Analysts, accountants, advisors, auditors | Financial modeling, audit trails, portfolio analysis |
| Trade/Service | Plumbers, electricians, contractors, inspectors | Code compliance, estimation, scheduling, diagnostics |
| Real Estate | Agents, property managers, appraisers | Market analysis, lease negotiation, valuation |
| Healthcare Admin | Office managers, billing specialists, practice coordinators | Scheduling, insurance, patient communication |
| Journalism | Reporters, editors, fact-checkers | Source verification, investigative research, editorial standards |

Each Operator engine is a YAML configuration: governance profile (domain-specific integrity rules), recommended tools/skills, workbench layout, and onboarding path. Developers create new engines without touching core platform code. The Talent Marketplace means Operators publish domain-specific Talents — the displaced teacher's curriculum checker becomes a $29/month subscription that 50 schools use.

### Knowledge Engines (Cross-Cutting — Serves Both Creators and Operators)

Knowledge engines don't produce final creative output — they organize, connect, and surface intelligence that feeds into creative and operator workflows. Both the novelist and the clinical researcher need structured knowledge management.

| Engine | Who It Serves | Governance Profile | Readiness |
|--------|-------------|-------------------|-----------|
| **Knowledge** | Second-brainers, PKM users (Obsidian, Logseq, Notion, Readwise) | Source Provenance | Partially built |
| **Research** | Academics, journalists, analysts, anyone doing deep investigation | Citation Integrity | Planned |

### Life Data Engines (Feeder — Provide Intelligence to All Other Engines)

Life Data engines don't produce external output. They capture signals about the creator/operator's life and feed that intelligence into every other engine. The life-data two-tier taxonomy identifies two layers: Level 1 (Body — physical existence, routines, location) and Level 2 (Mind — creative consumption, reading, entertainment, social patterns).

| Engine | Who It Serves | Governance Profile | Readiness |
|--------|-------------|-------------------|-----------|
| **Body** | Health signal users (Fitbit, Apple Watch, Oura, Whoop) | Health Signal Integrity | Spec ready |
| **Perception** | Visual capture users (smart glasses, camera, screenshots) | Visual Context Validation | Future |
| **Consumption** | Netflix, Kindle, Audible, Goodreads, Pocket users | Consumption Pattern Integrity | Spec ready (LD-1 through LD-8) |

The Life Data Correlation Engine (LD-8) is the highest-leverage component: it cross-references body signals, reading patterns, entertainment consumption, location, and creative output to give the Alter Ego a complete picture of how the creator's life inputs map to creative outputs. "You write best in the mornings after good sleep. You consume memoir content before starting memoir projects. Your current patterns suggest you're entering a creative phase."

### Custom Engines (Developer-Created — Unbounded)

The architecture supports arbitrary engine creation via engine YAML definitions. No Python code needed for basic engines. A developer creates a YAML file defining governance profile, component selection, workbench layout, source/sink affinity, and onboarding path — restarts the platform — and the engine appears in the selector.

Marketplace users share entire engine configurations as installable recipes: "My Newsletter Engine," "My Clinical Research Engine," "My YouTube Shorts Engine," "My RPG Writer's Engine." Engines become shareable creative environments, not just workflows.

All engines share the same foundation: Alter Ego (memory), Governor (quality), Codex (knowledge), Tournaments (selection), Crystallizer (learning), BYOK Routing (model access), Makers' Workbench (UI shell).

## Agent-C Exclusive Capabilities

These exist only in the paid app layer. Life-OS users get the same engine but without these experience features.

### 1. Capacitor Native App

The product. One React codebase, three platforms.

| Feature | Status | Details |
|---------|--------|---------|
| iOS app | Spec complete | App Store ready, native plugins (camera, haptics, Face ID, push) |
| Android app | Spec complete | Google Play ready, native plugins (haptics, biometrics, push) |
| Web SPA | Built | React Workbench, responsive |
| Voice-first mobile input | Spec complete | Large mic button, native speech recognition, offline queue |
| Push notifications | Designed | Morning briefing, Governor alerts, crystallization done, milestone |
| Cross-device continuity | Spec complete | Start on phone, continue on desktop |

Estimated build: ~13 dev days (M-0 through M-6).

### 2. Makers' Workbench

The central product UI. Everything opens from within it.

| Panel | Status | What It Does |
|-------|--------|-------------|
| Source Valve Panel | Built | Left wall — connected sources with state indicators (dormant/connected/active/error) |
| Sink Valve Panel | Built | Right wall — publishing destinations with state indicators |
| Skills Library | Built | Grid of available Talents with state badges (available/learning/ready/mastered) |
| Recipe Canvas | Built | Center — visual skill wiring with flow animation |
| Ingredients Bar | Built | Bottom — active CodexChunks and sources as interactive pills |
| Alter Ego Avatar | Built | Floating presence with name, growth stage, status indicator |
| Alter Ego Chat | Built | Expandable chat overlay for creative conversation |
| Active Recipes | Built | Recent projects, quick access to continue |

### 3. Alter Ego Full UI

The digital twin — the moat. Backend exists for everyone; the UI is Agent-C exclusive.

| Feature | Status | Details |
|---------|--------|---------|
| Growth stage badge | Built | Seedling → Growing → Rooted → Blooming → Sovereign |
| Mirror Reflection | Built | MirrorReflection.tsx — creative development reflection |
| Provenance Scores | Built | ProvenanceScoreCard.tsx — weighted composite showing creative DNA |
| Growth Edges display | Built | GrowthEdgesList.tsx — 5 edge patterns (exploration, mastery, expression, structural, community) |
| Agent name picker | Built | PersonaNamePicker.tsx — creator names their Alter Ego ("Muse", "Scribe", custom) |
| Overnight crystallization UI | Built | Visual indicator of what was learned from conversations while creator slept |

### 4. Creator Development Engine (CDE) Dashboard

Tracks every creative decision, reveals growth patterns.

| Component | Status | What It Shows |
|-----------|--------|-------------|
| Decision event tracking | Built | 19 decision types across 5 services |
| CreatorProfileCard | Built | Composite view of creative DNA |
| Decision density signals | Built | Active engagement weighting (30% active + 70% passive) |
| Growth edge detection | Built | 5 patterns: exploration, mastery, expression, structural, community |
| /mirror command (Telegram) | Built | Creative development reflection with ASCII charts |
| /provenance command | Built | Score breakdown with visual indicators |
| REST endpoints | Built | GET /creator/profile, /provenance, /mirror |

### 5. Visual Governor Dashboard

Narrative governance made visible with inline resolution.

| Feature | Status | Details |
|---------|--------|---------|
| N1-N5 violation display | Built | Color-coded severity (CRITICAL red, HIGH orange, MEDIUM yellow) |
| Kill switch indicators | Built | Arc Wholeness, Character Integration, Setup-Payoff Distance |
| ORA loop display | Built | Observe → Reason → Act progression with cycle count |
| GovernorBoundary component | Built | Boundary overlay for spatial canvas |
| Inline resolution | Designed | Accept/reject Governor suggestion directly in editor |

### 6. Agent M — Daily Briefing Intelligence

AI-powered personal creative intelligence officer.

| Feature | Status | Details |
|---------|--------|---------|
| Morning briefing | Built | BriefingPanel.tsx — project status, recent crystallizations, Governor flags |
| Mission flow | Built | Recommended next action based on pipeline stage + CDE signals |
| Intel panel | Built | IntelPanel.tsx — new connections, knowledge discoveries, growth signals |
| Suggested actions | Built | SuggestedActions.tsx — context-aware next steps |
| Push notification delivery | Designed | PushNotificationService — 8am daily briefing, Governor alerts |

### 7. One-Click OAuth Source Connectors

Connect data sources without managing API keys.

| Source | Status | Data Type |
|--------|--------|-----------|
| Google (Gmail/Drive/YouTube) | Built | Email, files, video metadata |
| Pinterest | Built | Pins, boards, aesthetic profiles |
| YouTube (Channel/Playlists) | Built | Video metadata, transcripts |
| Readwise | Built | OAuth API, Contextual Donut pattern (highlight + context + annotation) |
| Obsidian | Built | Reads vault filesystem directly (no export needed), wiki-link graph edges |
| Roam Research | Built | JSON upload, block-level crystallization with lineage breadcrumbs |
| Logseq | Built | LogseqNormalizer → reuses Roam pipeline (~30 LOC adapter) |
| TextNetworkService | Built | Louvain community detection, bridge insights, structural gap prompts (NetworkX) |
| Notion | Future | Databases, pages, properties |

### 8. One-Click Publish Sink Connectors

Publish to platforms without credential management.

| Sink | Tier | Status | Details |
|------|------|--------|---------|
| Substack | Creator+ | Built | Newsletter publishing |
| Medium | Creator+ | Built | Article publishing |
| YouTube | Creator+ | Built | Video upload with metadata |
| TikTok | Creator+ | Built | Short-form video via Content API |
| KDP (Kindle Direct Publishing) | Showrunner | Designed | EPUB export, cover generation |
| ElevenLabs Audiobook | Showrunner | Designed | TTS, M4B with chapter markers |

### 9. Rich Artifact Cards

Artifacts aren't JSON dumps. They're interactive cards with inline decisions.

| Artifact Type | Component | Inline Actions |
|---------------|-----------|---------------|
| Scene Draft | SceneCard.tsx | Accept / Reject / Revise / Export |
| Story Beat | BeatCard.tsx | Accept / Re-order / Extend / View connections |
| Character Profile | CharacterCard.tsx | Edit / Merge / Lock-in / Preview |
| Voice Sample | VoiceSampleCard.tsx | Play / Use this voice / Calibrate further |
| Governor Finding | GovernorCard.tsx | Accept fix / Reject / Explain / Override |
| Trailer Preview | TrailerCard.tsx | Watch / Share / Publish / Revise |

### 10. Talent Marketplace

A talent agency model for discovering, installing, and publishing capabilities.

| Feature | Tier | Status |
|---------|------|--------|
| Browse marketplace | Creator+ | Built |
| Search talents (Claude, OpenClaw, MCP sources) | Creator+ | Built |
| Install talent | Creator+ | Built |
| Rate talent | Creator+ | Built |
| Publish to marketplace | Showrunner | Built |
| Earn credits from published talents | Showrunner | Built |
| ACV risk tier display | Creator+ | Built |

### 11. Video Pipeline

Full video generation from story beats through rendering.

| Component | Tier | Status | Details |
|-----------|------|--------|---------|
| VideoDirectorService | Showrunner | Built | Orchestrates generation pipeline |
| VideoDraftService | Showrunner | Built | Draft generation with variants |
| VideoFinalService | Showrunner | Built | Final render from selected draft |
| VideoTournamentService | Showrunner | Built | Tournament selection for best variant |
| ShotPlan generation | Showrunner | Built | Scene → visual shot breakdown |
| 5 sink providers | Showrunner | Built | Higgsfield, Kling, Veo, Moltbook, Aggregator |
| Trailer generation | Creator+ | Built | 3-scene extraction from bookends, watermark |

### 12. Billing & Credit System

Transparent cost tracking with audit trail.

| Component | Status | Details |
|-----------|--------|---------|
| Paddle integration | Built | Checkout, webhooks, customer portal |
| Credit ledger | Built | Double-entry, append-only, per-tenant |
| Budget orchestrator | Built | Routes to provider, checks balance, deducts credits |
| Cost dashboard | Built | BudgetStatusWidget, per-service breakdown |
| Tier gate middleware | Built | 402 PAYMENT_REQUIRED with X-Required-Tier header |
| Usage limit enforcement | Built | SurfaceAccessService controls scenes/mo, video renders/mo |

### 13. Managed Hosting

Creators don't manage servers.

| Component | Status | Technology |
|-----------|--------|-----------|
| Backend | Deployed | Cloud Run (FastAPI) |
| Database | Managed | Cloud SQL (PostgreSQL) |
| Auth | Built | Firebase Authentication |
| LLM routing | Built | BudgetOrchestrator → Gemini/Claude/GPT |
| Auto-crystallization | Built | Overnight batch processing |
| Event sourcing | Built | Full audit trail, transactional outbox |

## Creative Refinement (Shared Engine, Premium UI)

The refinement loop runs in both Life-OS and Agent-C. The difference is how creators interact with it.

| Component | Engine (Both) | UI (Agent-C Only) |
|-----------|--------------|-------------------|
| Creative Refinement Orchestrator | Built — generic generate→critique→revise loop | Visual cycle indicator, inline accept/reject |
| Writing Critic | Built — N1-N5 audit wrapper | Governor dashboard with severity colors |
| Video Critic | Built — 5-dimension quality scoring | Frame-by-frame comparison cards |
| Exemplar Retriever | Built — corpus flywheel | "Similar to your best work" visual suggestions |
| Identity Lock Service | Built — 3 locking strategies | Character consistency indicator on scene cards |

## What Life-OS Users Get vs Agent-C Users

| Capability | Life-OS (Free) | Agent-C (Paid) |
|-----------|---------------|----------------|
| Engine (Governor, GraphRAG, Voice Cal, Story Bible, Codex) | Full access | Full access |
| Interface | CLI + Telegram/WhatsApp/Discord + MCP | Beautiful native app |
| Source connectors | API-level integration | One-click OAuth with visual valve panels |
| Sink connectors | API-level integration | One-click publish with preview |
| Alter Ego | Backend capabilities only | Full UI with growth stages, mirror, provenance |
| CDE (Creator Development Engine) | REST API only | Dashboard with visualizations |
| Governor | API audit response | Visual dashboard with inline resolution |
| Briefings | None | Agent M daily intelligence |
| Marketplace | None | Browse, install, publish, earn |
| Video pipeline | API access (BYOK) | Full UI with tournament selection |
| Hosting | Self-hosted or BYOK | Managed Cloud Run |
| Push notifications | None | Native mobile + browser |
| Voice input | None | Mic button with offline queue |

## Competitive Moats (Agent-C Specific)

1. **The Alter Ego** — Compounding creative intelligence. After 6 months, switching costs are enormous because the AI knows their voice, patterns, and decisions.
2. **The Narrative Governor** — Deterministic quality rules that no competitor enforces. 2,650+ tests of proven governance.
3. **The Disposable Software Economy** — Creators build niche tools via Agentic Engineering → publish to marketplace → network effects.
4. **The Creator Development Engine** — Only Agent-C tracks growth edges, provenance, and decision patterns.
5. **The Workbench** — Beautiful, intuitive, voice-first. Telegram is free but feels free. The Workbench is worth paying for.

## Implementation Status Summary

| Category | Built | Designed/Spec'd | Future |
|----------|-------|----------------|--------|
| Capacitor native app | Web SPA | iOS, Android, push, offline | — |
| Makers' Workbench | 9 panels | — | — |
| Alter Ego UI | 6 components | — | — |
| CDE Dashboard | 4 phases complete | CDE-5 deferred | — |
| Governor Dashboard | 3 components | Inline resolution | — |
| Agent M Briefing | 4 components | Push delivery | — |
| Source connectors | 8 (Google, Pinterest, YouTube, Gmail, Readwise, Obsidian, Roam/Logseq, TextNetwork) | — | 1 (Notion) |
| Sink connectors | 4 (Substack, Medium, YouTube, TikTok) | 2 (KDP, ElevenLabs) | — |
| Artifact cards | 6 types | — | — |
| Marketplace | Backend complete | UI partial | Credit cashout |
| Video pipeline | Full backend + 5 sinks | — | Gated on render quality |
| Billing | Full (Paddle) | — | — |
