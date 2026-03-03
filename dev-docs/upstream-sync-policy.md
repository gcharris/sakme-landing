# Upstream Sync Policy

> Defines how Life OS ingests changes from external code repositories.

## Definition

In this project, **upstream** means:

- external code repositories we pull implementation patterns or code from
- example: OpenClaw main repo and specific OpenClaw contributor repos

It does **not** mean:

- web research sources
- creator content sources (Drive, YouTube, URLs, etc.)
- product documentation references

## Why This Exists

Recent brain/loop improvements were informed by external agent-loop implementations. Without a sync policy, behavior drifts and security/quality risks increase.

## Sync Cadence

1. **Weekly scan** (low-cost): review upstream release notes/commits for loop/runtime changes.
2. **Monthly integration window** (batched): import selected changes behind feature flags.
3. **Hotfix exception**: critical bug/security fixes can bypass monthly window with Director approval.

## Intake Rules

Eligible for import:

- agent loop control flow
- retry/fallback patterns
- memory/compaction mechanisms
- tool execution orchestration patterns

Not imported directly:

- unvetted third-party integrations
- provider keys/config from upstream examples
- permissive logic that bypasses CES governance constraints

## Required Gates Before Merge

1. **Security/governance gate**
   - verify no bypass of policy/approval boundaries
   - verify tenant isolation remains intact
2. **Compatibility gate**
   - `python -m py_compile` on touched backend files
   - focused pytest for touched services
   - regression sweep on core agent tests
3. **Observability gate**
   - ensure failures are surfaced with logs/safe refs
   - confirm fallback paths are traceable in logs/events

## Traceability Requirements

Each upstream sync PR must include:

- upstream repo URL
- upstream commit/tag reference
- local branch + commit mapping
- summary of adopted vs rejected changes

## Rollback Strategy

- keep sync changes behind flags where possible
- define immediate rollback toggle in PR notes
- if behavior regresses, revert entire sync slice rather than partial hot edits

## Suggested PR Template Section

Use this section for upstream sync work:

```md
### Upstream Sync Metadata
- Upstream repo:
- Upstream commit/tag:
- Imported areas:
- Explicitly rejected areas:
- Feature flags/toggles:
- Validation commands run:
```
