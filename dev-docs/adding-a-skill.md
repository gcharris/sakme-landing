# Adding a Skill

> Current as-built patterns for adding a skill in Life OS core.

## Overview

Skill framework Layer 1 is now implemented with two active execution surfaces:

1. **DB-installed skills + audit trail** (`/api/v1/skills/*`, `Skill`, `SkillRun`, `SkillExecutor`)
2. **Code-registered workbench skills** (`backend/app/services/skills/registry.py`)

Use this guide to add skills that work with the current runtime.

## Current Skill Surfaces

### A) DB-installed skills (tenant-scoped)

Primary files:

- `backend/app/models/skill.py`
- `backend/app/models/skill_run.py`
- `backend/app/api/v1/skills.py`
- `backend/app/services/skill_executor.py`

Behavior:

- Skills are installed per tenant (`tenant_id`, `slug` uniqueness).
- Executions create `SkillRun` audit records.
- Local skill files can be integrity-checked via content hash.
- `SkillExecutor` currently enforces resolution + audit + integrity logic.
- Python/agentic generic execution paths are scaffolded and intentionally constrained (`NotImplementedError` placeholders indicate non-generic execution path).

### B) Code-registered skills (workbench wrappers)

Primary file:

- `backend/app/services/skills/registry.py`

Examples already registered:

- `format-picker`
- `engagement-analysis`
- `content-calendar`
- `inbox_router` (plus compatibility slug)
- `url_intelligence` (plus compatibility slug)

These wrappers expose a `slug` and async `run(inputs)` method and are used by skill-oriented application flows.

## How to Add a New Skill (Current Pattern)

### 1) Implement a skill class

Create a class under `backend/app/services/skills/` or the relevant service module:

```python
class MySkill:
    slug = "my-skill"

    async def run(self, inputs: dict | None = None) -> dict:
        payload = inputs or {}
        # business logic
        return {"success": True, "result": {...}}
```

### 2) Keep IO explicit

Use typed validation (Pydantic/dataclass) where input complexity warrants it. Return structured dict output with predictable keys (`success`, payload, and optional `skill_run` metadata).

### 3) Register the slug

Add the class to:

- `backend/app/services/skills/registry.py`

If needed, add backward-compatible alias slugs for existing clients.

### 4) Add/extend API wiring if externally callable

For REST access, add or update router handlers under `backend/app/api/v1/`.

### 5) Add tests

At minimum:

- input validation / error behavior
- successful run output shape
- slug registration test (if registry-backed)

## Security and Reliability Rules

1. If skill uses local file entry points, retain integrity hash checks.
2. Keep tenant isolation explicit in DB queries.
3. Use safe error handling in API surfaces (no raw stack leakage).
4. For LLM-backed behavior, prefer deterministic fallbacks where feasible.
5. For long-running or external side effects, use existing async/outbox patterns.

## Migration Path

Current practical path:

1. Continue wrapping concrete services as skill classes with stable slugs.
2. Expand DB-installed skill execution paths as executor backends are finalized.
3. Keep slug compatibility aliases during transition to avoid client breakage.

## Minimal Skill Checklist

Before merge:

- [ ] class has `slug`
- [ ] `run()` handles empty inputs safely
- [ ] registered in `services/skills/registry.py` (if code-registered path)
- [ ] tests cover success + bad input
- [ ] `python -m py_compile` clean on modified files

## Related Documents

- [adding-a-sink.md](adding-a-sink.md)
- [adding-a-source.md](adding-a-source.md)
- [adding-a-format.md](adding-a-format.md)
- [architecture-overview.md](architecture-overview.md)
