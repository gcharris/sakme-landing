# Narrative Governor

> Kill switches, N1-N5 failure taxonomy, ORA loop, and structural metrics.

## Overview

The Narrative Governor is the **deterministic quality gate** that prevents AI hallucination in creative output. LLMs cannot reliably maintain continuity, character consistency, or narrative structure across scenes. The Governor enforces these rules externally via the **N1-N5 failure taxonomy**, three **Kill Switches**, and an **ORA (Observe-Reason-Act) correction loop**.

Adapted from the Open-Ludic Stack's HATI protocol.

## Architecture

```
┌──────────────────────────────────────────────┐
│               ORA Loop                        │
│                                               │
│  OBSERVE ──► GovernorService.audit_scene()    │
│     │        (Check N1-N5 violations)         │
│     ▼                                         │
│  REASON  ──► Generate correction guidance     │
│     │        (What failed + how to fix)       │
│     ▼                                         │
│  ACT     ──► Regenerate with corrections      │
│     │                                         │
│     └──► Repeat until passed or max_iterations│
│                                               │
│  Post-scene: Run Kill Switches (warnings)     │
└──────────────────────────────────────────────┘
```

## Key Service

**File:** `backend/app/services/governor_service.py`

### GovernorService

```python
class GovernorService:
    async def audit_scene(self, scene_content, project_id, beat_id) -> GovernorAuditResult
    async def audit_and_correct(self, scene_content, project_id, beat_id, framework_id) -> ORAResult
```

The `audit_scene()` method runs the scene through N1-N5 checks and returns a structured result with violations, severity, and correction guidance. The `audit_and_correct()` method wraps this in an ORA loop that iterates up to `max_iterations` times.

## N1-N5 Failure Taxonomy

| Code | Type | Severity | Example | Detection |
|------|------|----------|---------|-----------|
| **N1** | Character Contamination | CRITICAL | Villain speaks hero's catchphrase | LLM audit |
| **N2** | Continuity Break | CRITICAL | Dead character appears alive | Graph query |
| **N3** | Arc Violation | HIGH | Payoff without setup | Graph query |
| **N4** | Voice Drift | HIGH | POV voice changes mid-scene | Embedding distance |
| **N5** | Pacing Failure | MEDIUM | 3 climaxes in a row | LLM audit |

### Detection Methods

| Method | Used For | How |
|--------|----------|-----|
| **LLM audit** | N1, N4, N5 | Send scene + context to LLM, ask for violations |
| **Graph query** | N2, N3 | Check Neo4j for dead characters, missing setups |
| **Embedding distance** | N4 (voice) | Cosine distance from VoiceProfile reference sample |

The hybrid approach (graph + LLM) is a deliberate design choice. Graph queries provide deterministic, fast checks for structural issues (N2, N3). LLM checks handle nuanced issues (N1, N4, N5) that require semantic understanding.

## Three Kill Switches

Post-scene structural checks that run after the scene is committed. These return warnings, not rejections.

| Kill Switch | Threshold | Measures |
|-------------|-----------|----------|
| **Arc Wholeness** | >= 3 beats/act | Fractal story structure completeness |
| **Character Integration** | >= 0.5 connectivity | No orphan characters in the graph |
| **Setup-Payoff Distance** | <= 5 scenes | Open setups must resolve within 5 scenes |

Kill switches are implemented in `StructuralMetricsService` and invoked from `DirectorService` after scene commit:

```python
kill_switches = await self.governor_service.structural_metrics_service.run_all(
    project_id=str(project_id),
    framework_id=framework_id,
)
```

## ORA Loop

The Observe-Reason-Act loop is the correction mechanism:

1. **Observe**: Audit the scene, collect N1-N5 violations
2. **Reason**: Generate correction guidance per violation (what failed, how to fix)
3. **Act**: Regenerate the scene with correction instructions injected into the prompt
4. **Repeat**: Until all violations resolved or max iterations reached

Each ORA iteration costs 3 LLM calls (audit + reason + act). The `DirectorService` tracks `ora_iterations` in the `DirectorResult`.

## Governor Configuration

Framework YAML files include governor-specific configuration:

```yaml
governor_config:
  n1_threshold: "critical"
  n2_threshold: "critical"
  n3_threshold: "high"
  n4_threshold: "high"
  n5_threshold: "medium"
  max_ora_iterations: 5
```

The `FrameworkService` loads this configuration and the Governor uses it to determine severity levels per framework. TV screenplays might have stricter N2 (continuity) thresholds than poetry, for example.

## Violation Schema

**File:** `backend/app/schemas/governor.py`

```python
class GovernorViolation(BaseModel):
    code: str          # "N1", "N2", etc.
    severity: str      # "critical", "high", "medium"
    description: str   # Human-readable explanation
    correction_guidance: str  # Specific fix instructions
    location: str | None     # Where in the scene

class GovernorAuditResult(BaseModel):
    passed: bool
    violations: list[GovernorViolation]
    scene_hash: str
```

## Audit Status Flow

Scenes persist with an audit status that tracks their governance journey:

| Status | Meaning |
|--------|---------|
| `PASSED` | All checks pass, scene is canonical |
| `FAILED_REJECTED` | Violations unresolved after ORA loop |
| `FAILED_ACCEPTED` | Director force-accepted despite violations |
| `ESCALATE_PARDON` | Governor granted provisional pardon |
| `PENDING` | Not yet audited |

Failed scenes are persisted to SQL (so they have an ID for force-accept) but are **not** written to Neo4j to avoid polluting the story graph.

## Integration with DirectorService

The Governor is Stage 5 of the 6-stage Director pipeline:

```
Stage 1: Scaffold Enrichment
Stage 2: Structure Variants
Stage 3: Scene Generation Tournament
Stage 4: Scoring + Voice Drift
Stage 5: Governor Auto-Audit (ORA Loop)  <-- Here
Stage 6: Graph Commit
```

## Related Documents

- [society-of-thought.md](society-of-thought.md) -- Tournament that feeds the Governor
- [creative-refinement.md](creative-refinement.md) -- Generic refinement loop inspired by ORA
- [knowledge-graph.md](knowledge-graph.md) -- Graph queries used by N2/N3 checks
