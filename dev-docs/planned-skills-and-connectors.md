# Planned Skills, Connectors & Capabilities

> **Purpose:** Ideas that have floated across brainstorms, research docs, tier specs, and conversations — consolidated into one prioritized list for launch planning.
> **Updated:** 2026-02-26
> **Sources:** Brain metaphor research, tier spec, Palantir Weekend memo, pipeline docs, brainstorming directory, Agency Era Manifesto
> **Rule:** When work begins on an item, create a task in WORK_INDEX and note the status here. When it ships, move to [skills-and-tools-built.md](skills-and-tools-built.md).

---

## How to Read This Document

Everything here is **an idea, not a commitment**. The Director decides what gets built. Items are organized by:

1. **Connectors** — New data sources (sensory organs)
2. **Skills** — New user-facing capabilities
3. **Tier-Specific Features** — Capabilities tied to specific subscription tiers
4. **Marketplace Seeds** — First-party skills that demonstrate the Marketplace pattern
5. **Marketplace Candidates** — Domain-specific skills the community should build (NOT us)

---

## 1. Planned Connectors (New Sensory Organs)

### Must Have — Launch Day

| Connector | Brain Analog | What It Does | Est. LOC | Source |
|-----------|-------------|-------------|----------|--------|
| **Notion** | Language (Wernicke's) | Block-level import with database/page structure. Second most popular PKM after Obsidian. Blocks researchers, teachers, consultants. | ~300 | Brain research §2.2 |
| **Google Calendar** | Memory (hippocampus) | Temporal context — when you work, when you meet, when you're free. Universal sensory organ. | ~150 | Brain research §2.2, Codex dispatch (CAL-1) |
| **CSV Import (enhanced)** | Various | Generic CSV with auto-detect for Netflix, Kindle, Audible, Goodreads. Gateway to all consumption data. | ~200 | Brain research §2.2, Codex dispatch (CSV-1) |

### Should Have — Month 1

| Connector | Brain Analog | What It Does | Est. LOC | Source |
|-----------|-------------|-------------|----------|--------|
| **Netflix viewing history** (LD-2) | Taste (insula) | Entertainment consumption patterns. "The system knows what I watch." Immediate delight. | ~80 | Brain research §2.2, tier spec |
| **Kindle highlights** (LD-3) | Taste (insula) | Reading patterns — the richest creative signal. Direct Alter Ego taste mapping. | ~100 | Brain research §2.2, tier spec |
| **Apple Notes** | Language (Wernicke's) | Most used note-taking app by install base. The "I'm not technical" user's knowledge store. | ~200 | Brain research §2.2 |
| **Pocket / Instapaper** | Taste (saved intent) | Read-later = curated intent. Researchers and journalists rely on these. Already have Pocket service — Instapaper is incremental. | ~120 | Brain research §2.2 |
| **Browser Bookmarks (Chrome)** | Memory (hippocampus) | Curated intent archive. What you saved to return to. | ~100 | Brain research §2.2 |

### Nice to Have — Quarter 1

| Connector | Brain Analog | What It Does | Est. LOC | Source |
|-----------|-------------|-------------|----------|--------|
| **Audible library** (LD-4) | Hearing (auditory cortex) | Completes the consumption trinity (Netflix + Kindle + Audible). | ~80 | Brain research §2.2 |
| **Spotify / Apple Music** | Hearing (auditory cortex) | Aesthetic signal — what you listen to while creating. Mood mapping. | ~150 | Brain research §2.2 |
| **GPS / Location** (LD-5) | Proprioception (somatosensory) | Location-creativity correlation. "You write best at coffee shops." | ~120 | Brain research §2.2 |
| **Social History** (LD-6) | Social perception (fusiform) | Twitter archive, LinkedIn. Public voice analysis. | ~100 | Brain research §2.2 |
| **Google Docs (deep)** | Language (Wernicke's) | Beyond Drive listing — collaborative editing history, comments, suggestion trails. | ~250 | Brain research §2.2 |
| **Amazon purchase history** | Memory (hippocampus) | What you bought reveals interests. Temporal consumption anchors. | ~100 | Brain research §1.1 |
| **Evernote** | Language (Wernicke's) | Legacy PKM migration. Many long-time users trapped here. | ~200 | Brain research §3.2 |
| **Tana** | Language (Wernicke's) | Emerging PKM. Small but passionate user base. | ~150 | Pipeline docs |
| **Apple Health** | Proprioception | iOS health data (complement to Fitbit/Oura). | ~150 | Tier spec |
| **Whoop** | Proprioception | Performance/recovery tracking for fitness-focused users. | ~100 | Brain research §1.1 |

---

## 2. Planned Skills (New Capabilities)

### Must Have — Launch Day

| Skill | What It Does | Est. LOC | Source | Status |
|-------|-------------|----------|--------|--------|
| **Alter Ego Blueprint** | Conversational Alter Ego creation — guided flow that builds digital twin through questions, not forms. | ~200 | Tier spec, Codex dispatch (BLUEPRINT-1) | Dispatched to Codex |
| **Template/SOP Builder** | Import a checklist or SOP, convert it to a governed Talent. Minimum Operator experience. Without this, Path B of the Manifesto is empty. | ~300 | Brain research §2.2, Manifesto |  |
| **Nightly Council** | Three-perspective AI debate (Growth, Revenue, Ops) producing prioritized recommendations. | ~250 | Tier spec, council.py schemas exist, Codex dispatch (COUNCIL-1) | Dispatched to Codex |
| **Morning Briefing** | Lightweight daily touchpoint — recent activity, upcoming calendar, growth edge, streak. No LLM call (free-tier safe). | ~150 | Brain research §2.2, Codex dispatch (BRIEF-1) | Dispatched to Codex |
| **Manuscript Export** | Clean Markdown + EPUB from project scenes. Writers must get work OUT. | ~200 | Brain research §2.2, Codex dispatch (EXPORT-1) | Dispatched to Codex |

### Should Have — Month 1

| Skill | What It Does | Est. LOC | Source |
|-------|-------------|----------|--------|
| **Reading Tracker** (LD-7) | Unified cross-medium reading profile. Depends on Kindle + Audible connectors. | ~120 | Brain research §2.3 |
| **Architecture Memory** | Persistent project context for Vibe Coder use case. System remembers your codebase across sessions. | ~200 | Brain research §2.2 |
| **Batch Operations** | Process 50 notes / 100 emails / entire vault at once. Power users need this within first week. | ~150 | Brain research §2.2 |
| **Scheduled Crystallization** | Cron job that auto-crystallizes new data from connected sources overnight. "The default mode network." | ~100 | Tier spec (Making tier, "habits") |
| **Creative Threat Detection** | Amygdala-like system: "Your voice consistency dropped 40% this week. Are you okay?" Burnout detection, quality decline alerts. | ~200 | Brain research §1.3 (amygdala gap) |
| **Cross-Engine Signal Surfacing** | Feeder engine signals visible as passive context at all tiers. Health data shows as subtle signal in Writer's Engine. "The corpus callosum." | ~150 | Brain research §1.3 (corpus callosum problem) |
| **LaTeX / Academic Export** | Export to LaTeX for researchers and academics. | ~100 | Brain research §3.2 |
| **Scrivener Export** | Export in Scrivener format for novelists. | ~80 | Brain research §3.2 |

### Nice to Have — Quarter 1

| Skill | What It Does | Est. LOC | Source |
|-------|-------------|----------|--------|
| **Consumption → Production Correlation** (LD-8) | Cross-domain analysis: "You write better after watching noir films." "Your health data correlates with creative output." | ~250 | Tier spec (Mastery tier), brain research §1.2 |
| **Podcast Engine** | Engine YAML for podcast production — outline, script, talking points, show notes. | ~200 | Brain research §3.2 |
| **Newsletter Engine** | Enhanced newsletter pipeline — weekly schedule, Substack integration, audience growth tracking. | ~150 | Brain research §3.2 |
| **RPG Writer Engine** | Engine YAML for tabletop RPG content — world building, encounter design, NPC generation. | ~200 | Brain research §3.2 |
| **YouTube Shorts Engine** | Optimized for short-form vertical video — hook analysis, trend integration, CTA templates. | ~150 | Pipeline docs |
| **Domain-Specific Onboarding Paths** | "I'm a clinical researcher" → custom 3-question flow instead of generic 8-screen canvas. | ~200 | Brain research §3.2 |
| **Marketplace Publishing Wizard** | Guided flow: write skill → test → set price → publish to Marketplace. | ~300 | Tier spec (all tiers can publish) |

---

## 3. Tier-Specific Features (Planned)

### Sense Tier ($0) — Already generous, but these make it complete:

| Feature | Status | Notes |
|---------|--------|-------|
| All engines, all connectors, all sinks | ✅ Decided | Everything works at free tier |
| Full skill library (cheap skills free) | ✅ Decided | Compute-intensive skills require credits |
| ClosedClaw verification free | ✅ Decided | Security is not a premium feature |
| BYOK at every tier | ✅ Decided | Bring your own Gemini/Claude/GPT key |
| 1 free Marketplace install | ✅ Decided | Sample the ecosystem |
| Marketplace publishing | ✅ Decided | Everyone can publish and earn credits |
| Text Network "wow moment" | Planned | First-session knowledge cluster visualization |
| N1-N5 Governor audit on paste | Planned | Paste a chapter → instant quality audit |

### Making Tier (~$9) — Habit formation:

| Feature | Status | Notes |
|---------|--------|-------|
| Hosted models (no BYOK required) | ✅ Decided | Platform pays for compute |
| Alter Ego (full digital twin) | ✅ Decided | Conversation crystallizer, persistent memory |
| Nightly habits (cron automation) | Planned | Scheduled crystallization, morning briefing |
| Recipe composition | ✅ Built | Skills compose into multi-step Recipes |
| Pattern detection (habit detector) | ✅ Built (backend) | Needs skill wrapper |

### Meaning Tier (~$19-29) — Emotional drive:

| Feature | Status | Notes |
|---------|--------|-------|
| Proactive nudging | ✅ Built (backend) | proactive_notification_service.py |
| Cross-domain correlation | Planned | "Your sleep affects your writing quality" |
| Advanced analytics / signal dashboard | ✅ Built (backend) | signal_aggregator_service.py needs UI |
| Intelligence summaries | ✅ Built (backend) | intelligence_summary_service.py needs skill wrapper |
| Creative threat detection (amygdala) | Planned | Voice consistency, quality decline, burnout |

### Mastery Tier (~$39-49) — Executive function:

| Feature | Status | Notes |
|---------|--------|-------|
| Nightly Council | Planned / dispatched | council.py schemas exist, COUNCIL-1 dispatched |
| Marketplace publishing (revenue) | ✅ Decided | Earn credits, crypto off-ramp future |
| Full API access | ✅ Decided | Programmatic access to all capabilities |
| Consumption-Production Correlation (LD-8) | Planned | The prefrontal cortex feature |
| Multi-engine orchestration | Planned | Writer + Filmmaker + Body working together |

---

## 4. Marketplace Seeds (First-Party Demonstration Skills)

These are skills WE build to demonstrate the Marketplace pattern. They show what community-built skills should look like.

| Seed Skill | Domain | Why We Build It | Est. LOC |
|------------|--------|----------------|----------|
| **Newsletter Template** | Writer's Engine | Basic newsletter YAML + Substack sink. Community creates "Morning Brew Engine", "Academic Newsletter Engine". | ~150 |
| **YouTube Script Template** | Filmmaker Engine | Script structure for YouTube videos. Community creates niche variants (cooking, tech, ASMR). | ~150 |
| **Workout Tracker** | Body Engine | Simple exercise logging + pattern detection. Community creates sport-specific trackers. | ~200 |
| **Invoice Generator** | Operator Engine | Basic invoice from templates. Community creates industry-specific variants. | ~150 |
| **Journal Prompt** | Writer's Engine | Daily journaling skill with reflection prompts. Community creates therapeutic, gratitude, creative variants. | ~100 |

---

## 5. Marketplace Candidates (Community Builds These — NOT Us)

These require domain expertise we don't have. They validate the Marketplace thesis.

| Skill | Domain | Why Not Platform | Who Builds It |
|-------|--------|-----------------|---------------|
| Clinical trial protocol validator | Pharma/Biotech | Regulatory domain knowledge | Ex-pharma researchers |
| Curriculum alignment checker | Education | Standards knowledge (Common Core, state-specific) | Teachers, curriculum designers |
| Legal contract reviewer | Legal | Liability, jurisdiction-specific | Lawyers, paralegals |
| Financial model validator | Finance | GAAP/IFRS expertise | Financial analysts |
| Real estate comp analyzer | Real Estate | MLS data, local markets | Real estate agents |
| Plumbing/electrical code checker | Trades | Code varies by municipality | Licensed tradespeople |
| Journalism source verifier | Media | Fact-checking methodology | Journalists |
| Music theory analyzer | Music | Harmonic analysis, genre classification | Musicians, composers |
| Academic citation formatter | Research | Style guide expertise (APA, MLA, etc.) | Researchers, librarians |
| SEO content optimizer | Marketing | Algorithm knowledge, keyword research | SEO specialists |
| Fitness programming | Health | Exercise science, periodization | Certified trainers |
| Recipe nutrition analyzer | Food | USDA database, dietary requirements | Nutritionists |
| Tax preparation assistant | Finance | Tax code varies annually + by region | CPAs, tax preparers |
| Immigration form helper | Legal | Immigration law expertise | Immigration lawyers |

---

## 6. Engine YAMLs (Planned)

Engine definitions that expand Life-OS beyond creative writing.

| Engine YAML | Domain | Status | Source |
|-------------|--------|--------|--------|
| `writer.yaml` | Creative writing | ✅ Built (multiple: novel, memoir, article, short_form_prose) | |
| `filmmaker.yaml` | Video production | ✅ Built (tiktok, youtube, family_video, prestige_drama, sitcom) | |
| `comic.yaml` | Comics/graphic novels | ✅ Built | |
| `podcast.yaml` | Podcast production | Planned | Brain research §3.2 |
| `newsletter.yaml` | Newsletter publishing | Planned (basic exists) | Brain research §3.2 |
| `operator.yaml` | SOP/checklist/process | Planned | Manifesto Path B |
| `vibe_coder.yaml` | Software development | Planned | Tier spec |
| `researcher.yaml` | Academic/clinical research | Planned | Brain research §2.2 (pharma persona) |
| `rpg_writer.yaml` | Tabletop RPG content | Planned | Brain research §3.2 |
| `youtube_shorts.yaml` | Short-form vertical video | Planned | Pipeline docs |

---

## Summary: What We Need for Launch

| Category | Count | Total LOC |
|----------|-------|-----------|
| **Must-Have Connectors** | 3 (Notion, Calendar, CSV) | ~650 |
| **Must-Have Skills** | 5 (Blueprint, SOP, Council, Briefing, Export) | ~1,100 |
| **Backend → Skill Conversions (launch)** | 5 (batch crystallize, contradiction, briefing, export, crystallizer UX) | ~540 |
| **Total launch-critical new work** | 13 items | ~2,290 LOC |

Of these, **5 are already dispatched to Codex tonight** (Calendar, CSV, Briefing, Export, Blueprint, Council) = ~1,100 LOC. That leaves ~1,190 LOC of new work: Notion connector, SOP builder, batch crystallize wrapper, contradiction wiring, crystallizer UX.

### What's Already Been Dispatched (Tonight's Codex Batch)

| Task | Covers |
|------|--------|
| TIER-1 | Subscription enum rename (Sense/Making/Meaning/Mastery) |
| CAL-1 | Google Calendar connector |
| CSV-1 | CSV Import Service |
| EXPORT-1 | Manuscript export |
| BRIEF-1 | Morning Briefing |
| COUNCIL-1 | Nightly Council |
| BLUEPRINT-1 | Alter Ego Blueprint |
