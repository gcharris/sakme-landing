# Skill Write Boundaries

> **Authority:** R-013.e (Director ruling 2026-02-22)
> **Enforcement:** Merge gate — PRs that bypass designated services get flagged in review.
> **ADR:** [`docs/plans/2026-02-22-skill-ecs-transition-adr.md`](../plans/2026-02-22-skill-ecs-transition-adr.md)

---

## The Rule

The `Skill` model carries fields for multiple concerns (execution, governance, marketplace, automation). To prevent monolith drift while the table remains denormalized, each concern has a **designated service owner**. Only that service may write to its fields.

This is enforced in code review, not at the ORM level. Component facades are part of the R-013 Phase A plan; until those land, enforce boundaries through designated services and PR review.

---

## Ownership Table

| Concern | Fields (indicative) | Owning Service(s) | Facade Status |
|---------|---------------------|--------------------|--------|
| **Identity** | `name`, `slug`, `description`, `version`, `source_url`, `source_format`, `publisher` | `SkillImportAdapter` + skills import endpoints | Planned (R-013.a) |
| **Execution** | `entry_point`, `is_active`, `last_run_at`, `run_count`, `last_error` | `SkillExecutor`, run audit flow | Planned (R-013.a) |
| **Governance** | `governor_profile`, `status`, `verified_at`, `config.acv_attestation` | `AcvPipeline`, `TalentGraduationService` | Planned (R-013.a) |
| **Marketplace** | `marketplace_status`, `marketplace_description`, `price_credits`, `revenue_split`, `install_count`, `rating_sum`, `rating_count`, `author_tenant_id` | `MarketplaceService` | Planned (R-013.a) |
| **Automation** | `config` metadata only (skill-level), plus linked records in `habits` / `standing_orders` | `HabitExecutor`, `StandingOrdersService` | Planned (R-013.a) |

---

## What This Means in Practice

**Allowed:**
```python
# MarketplaceService publishing a skill
skill.marketplace_status = "PUBLISHED"
skill.price_credits = max(0, int(price_credits))
db.commit()
```

**Not allowed:**
```python
# Some random endpoint directly setting marketplace fields
skill.marketplace_status = "PUBLISHED"  # VIOLATION — bypasses MarketplaceService
skill.install_count += 1  # VIOLATION — only MarketplaceService writes this
db.commit()
```

**Not allowed:**
```python
# SkillExecutor touching governance fields
skill.config["acv_attestation"] = {"verdict": "pass"}  # VIOLATION — only ACV/graduation services write this
```

---

## Review Checklist (for PR authors and reviewers)

Before merging any PR that touches the `Skill` model or `skills` table:

1. **Which fields are being written?** Map each field to its owning concern from the table above.
2. **Is the write happening through the designated service?** If not, it's a boundary violation.
3. **Is a facade being used?** Prefer `Component.from_skill()` / `component.apply_to(skill)` over direct attribute assignment.
4. **Does the PR introduce a new field on `Skill`?** If yes, assign it to a concern and document which service owns it.
5. **Does the PR create a new write path to existing fields?** If a second service now writes to the same concern, flag it — this may trigger Phase B decomposition criteria.

---

## Phase B Triggers (When to Decompose)

The `skills` table stays denormalized until **any 2** of these are true:

- `skills` receives another cross-cutting field group
- More than 3 services write disjoint field sets on `Skill`
- Migration churn: >2 releases touching `skills` schema in a single month
- Query performance: any API endpoint does `SELECT *` on `skills` to read a single concern (e.g., marketplace fields when only execution state is needed)

Track these metrics. When 2 trigger, file an issue for table decomposition per the ADR Phase B plan.

---

*Filed by: Cowork | 2026-02-22 | R-013.e*
