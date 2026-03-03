# Inbox Router

> File classification and routing skill for uploaded/imported assets.

## Where It Lives

- `backend/app/services/inbox_router_skill.py`
- `backend/app/services/inbox_router_telegram.py`
- `backend/app/services/skills/registry.py`

## Skill Identity

- class: `InboxRouterSkill`
- slug: `inbox_router`
- compatibility alias: `inbox-router`

## What It Does

`InboxRouterSkill.route(file_path, tenant_id)`:

1. detect MIME via `mimetypes.guess_type`
2. apply direct MIME map
3. apply prefix map (`image/`, `video/`, `audio/`, `text/`)
4. if unknown and LLM client exists, perform LLM classification fallback
5. return `RouteResult`:
   - `category`
   - `confidence`
   - `target_service`
   - `metadata`
   - `llm_classified`

## Route Targets

Current route map:

- photo -> `media_pipeline`
- video -> `video_preprocessor`
- audio -> `audio_preprocessor`
- document -> `crystallizer`
- data -> `crystallizer`
- unknown -> `inbox`

## Extension and MIME Coverage

Includes common office/media fallbacks:

- `.docx`, `.pptx`, `.xlsx`
- `.webm`, `.m4a`, `.flac`, `.ogg`

## Telegram Handler

`handle_document(...)` in `inbox_router_telegram.py`:

- downloads attachment to `/tmp`
- runs skill route
- always cleans temp file in `finally` (`unlink(missing_ok=True)`)
- replies with detected category + suggested route

## Operational Notes

- LLM classification is optional and only used for ambiguous files.
- For deterministic environments, instantiate without LLM client.
- Keep route map aligned with available downstream services.

## Quick Verification

```bash
cd /Users/gch2024/Dev/context-engine-studio/backend
python -m py_compile app/services/inbox_router_skill.py app/services/inbox_router_telegram.py
pytest tests/services/test_inbox_router_skill.py -v
```
