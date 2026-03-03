# Messaging Guardrails: Creative Physics Language Policy

> **Authority:** R-013.d (Director ruling 2026-02-22)
> **Scope:** All user-facing copy — UI, marketing, onboarding, docs, landing pages, social
> **Review gate:** Director approves final copy policy before launch
> **ADR:** [`docs/plans/2026-02-22-skill-ecs-transition-adr.md`](../plans/2026-02-22-skill-ecs-transition-adr.md)

---

## The Principle

The game engine architecture is real and structurally useful — ECS, scene composition, procedural generation all inform how the system works internally. But the user-facing experience must feel like a creative practice, not a video game. The game is underneath. The experience is above. The creator never needs to know.

---

## Approved Language

These phrases and framings are cleared for user-facing use:

| Context | Approved Phrasing |
|---------|-------------------|
| **Core positioning** | "Your creative practice has structure, memory, and momentum." |
| **Brand tagline** | "Agent-C gives your creative practice physics." |
| **Alter Ego value** | "We learn how you work." / "Your Alter Ego remembers." |
| **Habit formation** | "Your practice deepens over time." / "Repeated use becomes personalized practice." |
| **Workbench adaptation** | "Your workspace adapts to how you create." |
| **Talent progression** | "Skills earn trust through use." / "Your talents grow with you." |
| **Marketplace** | "Share what you've built." / "Discover what other creators have made." |
| **ACV certification** | "Vetted and certified." / "Trusted by the community." |
| **Governance** | "Quality guardrails, not creative limits." |
| **Open source (Life-OS)** | "The path is free. The monastery costs money." |

---

## Forbidden Language

These phrases and patterns must **never** appear in user-facing contexts. They trigger gamification associations that undermine the practitioner dignity the brand depends on.

| Forbidden | Why | Say Instead |
|-----------|-----|-------------|
| "Level up" / "leveling" | RPG progression framing | "Deepen" / "grow" / "progress" |
| "XP" / "experience points" | Explicit game mechanic | (no equivalent — don't quantify practice as points) |
| "Leaderboard" / "ranking" | Competition framing | "Community" / "fellow creators" |
| "Achievement badge" / "unlock" | Trophy/reward framing | "Milestone" / "capability" (if needed at all) |
| "Tutorial mode" / "Adventure mode" / "Sandbox mode" | Difficulty-mode pricing tiers | Use tier names directly: "Explorer" / "Creator" / "Showrunner" |
| "Quest" / "mission" / "challenge" | Game quest framing | "Task" / "next step" / "suggestion" |
| "Skill tree" | RPG progression map | "Talent library" / "your capabilities" |
| "Inventory" / "loadout" | Game inventory framing | "Your tools" / "your workspace" |
| "Boss fight" / "final boss" | Game combat framing | (just don't) |
| "Spawn" / "respawn" | Game spawn mechanics | "Create" / "generate" / "draft" |
| "NPC" | Game character framing | "Collaborator" / "assistant" / role name |
| "Grind" / "grinding" | Repetitive game labor | "Practice" / "refine" / "iterate" |
| "Loot" / "drops" / "rewards" | Random reward framing | "Results" / "outputs" / "what you've made" |
| "Score" (as game score) | Game scoring | "Quality" / "confidence" (internal metrics stay internal) |
| "Player" | Game player framing | "Creator" / "practitioner" / "you" |

---

## Internal vs. External Vocabulary (Quick Reference)

| Internal (architecture docs, agent chat, PRs) | External (UI, marketing, docs) |
|-----------------------------------------------|-------------------------------|
| Entity | Talent / Skill |
| Component | (not exposed) |
| System | (not exposed) |
| Scene composition | "Your workspace" |
| Procedural generation | "Adapts to how you work" |
| ECS refactor | (never mention) |
| Phase B decomposition | (never mention) |
| Enlightenment Score | (deferred — T5 not shipped) |

---

## Launch Review Checklist

Before any user-facing copy ships, verify:

- [ ] No forbidden language from the table above appears anywhere in the copy
- [ ] No internal architecture terms (ECS, entity, component, system, facade) leak into user-facing text
- [ ] Pricing tiers use their proper names (Explorer/Creator/Showrunner), not difficulty metaphors
- [ ] The Alter Ego is described in practitioner terms, not game AI terms
- [ ] Quality metrics and scores are presented as confidence indicators, not game scores
- [ ] The taxonomy progression (Tools → Skills → Talents → Habits) uses natural language, not tier numbers
- [ ] The "creative physics" framing communicates consequence and memory, not game mechanics
- [ ] Director has approved the final copy (R-013.d requires this before launch)

---

*Filed by: Cowork | 2026-02-22 | R-013.d*
