# Creative Refinement Loop

> Protocol-based generic loop, critic/generator pairs, convergence tracking.

## Overview

The Creative Refinement Loop is a **generic orchestration pattern** that iterates: Generator produces output, Prescriptive Critic evaluates it and provides specific revision instructions, Generator revises, repeat until quality thresholds are met or max cycles reached.

This is the spiritual successor to the Narrative Governor's ORA loop, generalized to work with any creative output type (prose, video, comics) through Python's `Protocol` types.

## Architecture

```
┌─────────────────────────────────────────────┐
│       CreativeRefinementOrchestrator         │
│                                              │
│  Generator(prompt)                           │
│     │                                        │
│     ▼  output                                │
│  Critic(output, context) ──► CriticResult    │
│     │                         │              │
│     │  if not passed:         │              │
│     │  revision_instructions ─┘              │
│     ▼                                        │
│  Generator(prompt, revision_instructions)    │
│     │                                        │
│     ▼  repeat until converged or max_cycles  │
│  RefinementTrace (full history)              │
└─────────────────────────────────────────────┘
```

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/services/creative_refinement_orchestrator.py` | Generic loop engine |
| `backend/app/services/writing_critic.py` | Prose critic (wraps GovernorService) |
| `backend/app/services/video_critic.py` | Video quality critic (5 dimensions) |
| `backend/app/services/panel_critic.py` | Comics cross-panel critic (4 dimensions) |
| `backend/app/schemas/refinement.py` | Data models (ConvergenceConfig, CriticResult, etc.) |

## Protocol Types

The orchestrator uses Python's `Protocol` for structural subtyping. Any callable matching the protocol can be plugged in without inheritance:

```python
class GeneratorFn(Protocol):
    async def __call__(
        self,
        prompt: str,
        revision_instructions: dict[str, str] | None = None,
        **kwargs: Any,
    ) -> Any: ...

class CriticFn(Protocol):
    async def __call__(
        self,
        output: Any,
        context: dict,
    ) -> CriticResult: ...
```

This means any async function or method matching these signatures works as a Generator or Critic.

## CreativeRefinementOrchestrator

**File:** `backend/app/services/creative_refinement_orchestrator.py`

```python
class CreativeRefinementOrchestrator:
    def __init__(
        self,
        generator: GeneratorFn | Callable,
        critic: CriticFn | Callable,
        config: ConvergenceConfig | None = None,
        content_extractor: Callable[[Any], str] | None = None,
        on_refinement_complete: Callable[[RefinementTrace], Any] | None = None,
    ):

    async def refine(self, initial_prompt: str, context: dict) -> RefinementTrace:
```

### Refinement Loop

1. Generate initial output from `initial_prompt`
2. For each cycle (1 to `max_cycles`):
   - Critique current output
   - If critic passes and `early_stop_if_all_pass` is True, stop
   - Otherwise, extract revision instructions
   - Generate revised output with revision instructions
   - Record the cycle
3. Return `RefinementTrace` with full history

The `content_extractor` parameter handles heterogeneous output types -- for video it might extract a URL, for prose it extracts text content.

## Convergence Configuration

**File:** `backend/app/schemas/refinement.py`

```python
class ConvergenceConfig(BaseModel):
    max_cycles: int = 3           # Range: 1-10
    dimension_thresholds: dict[str, float] = {
        "identity": 0.7,
        "narrative_fidelity": 0.8,
        "craft": 0.6,
        "emotional_tone": 0.7,
        "structural_fit": 0.8,
    }
    early_stop_if_all_pass: bool = True
```

## CriticResult

The key innovation: critics provide **prescriptive revision instructions**, not just scores:

```python
class CriticResult(BaseModel):
    dimension_scores: dict[str, float]        # Per-dimension quality scores
    revision_instructions: dict[str, str]     # Per-dimension specific fixes
    passed: bool                               # Overall pass/fail
    summary: str                               # Human-readable summary
```

## Built-in Critics

### WritingPrescriptiveCritic

**File:** `backend/app/services/writing_critic.py`

Wraps the GovernorService N1-N5 audit into the CriticResult format:

| N-Code | Maps to Dimension | Penalty |
|--------|-------------------|---------|
| N1 (Character Contamination) | `identity` | 0.5 (critical) |
| N2 (Continuity Break) | `narrative_fidelity` | 0.5 (critical) |
| N3 (Arc Violation) | `structural_fit` | 0.3 (high) |
| N4 (Voice Drift) | `identity` | 0.3 (high) |
| N5 (Pacing Failure) | `structural_fit` | 0.15 (medium) |

Base score per dimension is 0.9. Penalties are subtracted per violation, floored at 0.0.

### VideoPrescriptiveCritic

**File:** `backend/app/services/video_critic.py`

Five dimensions for video quality:

| Dimension | What it Measures |
|-----------|-----------------|
| `identity_consistency` | Character looks match across shots |
| `narrative_fidelity` | Video tells the intended story |
| `aesthetics` | Visual quality, composition |
| `mood_alignment` | Matches the emotional target |
| `technical_quality` | Resolution, artifacts, timing |

Uses an injectable scorer callable for VLM (Vision Language Model) integration.

### PanelConsistencyCritic

**File:** `backend/app/services/panel_critic.py`

Four dimensions for comics:

| Dimension | What it Measures |
|-----------|-----------------|
| `character_consistency` | Characters look the same across panels |
| `world_consistency` | Setting details match |
| `visual_continuity` | Panel-to-panel flow |
| `narrative_flow` | Story reads correctly across panels |

## RefinementTrace

The complete history of the refinement process:

```python
class RefinementTrace(BaseModel):
    cycles: list[RefinementCycle]    # Full cycle-by-cycle history
    final_content: str               # The final refined output
    total_cycles: int                # How many iterations ran
    converged: bool                  # Did the critic pass?
    final_scores: dict[str, float]  # Last critic scores
```

Each `RefinementCycle` records `input_content`, `critic_result`, and `output_content` (None for final cycle).

## Supporting Services

| Service | File | Purpose |
|---------|------|---------|
| `ExemplarRetriever` | `backend/app/services/exemplar_retriever.py` | Retrieves high-scoring past outputs as reference |
| `VisualRetriever` | `backend/app/services/visual_retriever.py` | Retrieves visual reference frames |
| `VisualStylist` | `backend/app/services/visual_stylist.py` | Augments prompts with style enforcement |
| `IdentityLockService` | `backend/app/services/identity_lock_service.py` | Character identity preservation (3 strategies) |

### IdentityLockService Strategies

| Strategy | How it Works |
|----------|-------------|
| Seed locking | Fix random seeds for consistent generation |
| Reference anchoring | Attach reference images to prompts |
| Prompt anchoring | Inject character descriptions into every prompt |

## Usage Example

```python
orchestrator = CreativeRefinementOrchestrator(
    generator=my_scene_generator,
    critic=WritingPrescriptiveCritic(governor=governor_service).critique,
    config=ConvergenceConfig(max_cycles=3),
    content_extractor=lambda result: result.content,
)

trace = await orchestrator.refine(
    initial_prompt="Write a confrontation scene...",
    context={"project_id": project_id, "beat_id": beat_id},
)
```

## Related Documents

- [narrative-governor.md](narrative-governor.md) -- The ORA loop that inspired this pattern
- [society-of-thought.md](society-of-thought.md) -- Tournament that generates initial output
- [voice-calibration.md](voice-calibration.md) -- Voice profiles used by the writing critic
