# URL Intelligence

> Three-mode URL analysis skill for tenant insights, director memos, and triage.

## Where It Lives

- `backend/app/services/url_intelligence_skill.py`
- `backend/app/services/url_intelligence_context.py`
- `backend/app/api/v1/url_intelligence.py`
- `backend/app/services/skills/registry.py`

## Skill Identity

- class: `URLIntelligenceSkill`
- slug: `url_intelligence`
- compatibility slugs in registry: `url_intelligence`, `url-intelligence`

## API Endpoints

All mounted in router with explicit paths:

- `POST /api/v1/url-intelligence/run`
- `POST /api/v1/skills/url-intelligence/run` (legacy alias)
- `POST /api/v1/url-intelligence/analyze`
- `POST /api/v1/url-intelligence/memo`
- `POST /api/v1/url-intelligence/triage`

## Modes

1. **Run** (`run`)  
   Generic orchestrated execution path with optional codex sync.
2. **Ring 1** (`analyze`)  
   Tenant/project-aligned insight output (`URLInsight` shape).
3. **Ring 2** (`memo`)  
   Director-style spark/mirror/synthesis memo (`SovereignMemo` shape).
4. **Ring 3** (`triage`)  
   Returns options + connection summary for fast decisioning.

## Context Assembly

- Tenant context uses bounded brief assembly.
- Director context uses architecture/config/service inventory snippets.
- Context assembly failures are logged and degrade gracefully.

## Codex Sync Behavior

When enabled in run path:

- builds `CrystallizationCandidate`
- uses `TransformationType.LLM_EXTRACTION`
- uses `CrystallizationTrigger.SOVEREIGN_SYNC`
- sync failures are logged as warning and do not hard-fail URL analysis

## Error Handling

All API handlers return safe reference IDs for internal failures:

- no raw stack traces in response payload
- full exception logged server-side with reference token

## Quick Verification

```bash
cd /Users/gch2024/Dev/context-engine-studio/backend
python -m py_compile app/services/url_intelligence_skill.py app/services/url_intelligence_context.py app/api/v1/url_intelligence.py
pytest tests/services/test_url_intelligence_skill.py -v
```
