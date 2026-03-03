# Society of Thought

> Tournament architecture, variant generation, consensus voting, and the DirectorService pipeline.

## Overview

CES externalizes the multi-agent reasoning pattern described in Kim et al. (arXiv:2601.10825). Instead of relying on a single LLM pass, the system generates **multiple variants** across different writing strategies, scores them via LLM-as-judge, and lets human selection (or automated ranking) pick the winner.

The core insight: high-performing reasoning models simulate internal multi-agent debates. CES externalizes and controls this dynamic through tournaments.

| Internal (Model) | External (CES) |
|------------------|----------------|
| Multiple simulated personas debate | Tournaments generate N variants |
| Conflict/reconciliation neurons | Rubric scoring + human selection |
| "Wisdom of crowds" in one pass | Director Mode across passes |
| Shared context across personas | Knowledge Graph as shared memory |

## The 6-Stage Director Pipeline

**File:** `backend/app/services/director_service.py`

The `DirectorService` implements the complete Society of Thought pipeline:

```
Stage 1: Scaffold Enrichment
    Load beat, project, voice profile, characters, continuity, research
    Build immutable SceneScaffold

Stage 2: Structure Variants
    LLM generates 3-5 structural approaches (Gemini at temp 0.4)
    Each has: beat_sequence, pacing_notes, strategic_rationale

Stage 3: Scene Generation Tournament
    Concurrent generation across all WritingStrategies
    Each variant gets: random temperature, strategy focus, voice constraints
    Wildcard (Move-37) variants injected at configurable ratio

Stage 4: Scoring + Voice Drift
    LLM-as-judge scores 6 criteria (0-100)
    Voice drift via embedding cosine distance
    Optional: Canon Energy + Big Idea alignment scoring

Stage 5: Governor Auto-Audit (ORA Loop)
    N1-N5 violation checks + correction iterations

Stage 6: Graph Commit
    SQL + Neo4j + EventStore triple-write
```

## Stage 1: Scaffold Enrichment

The `enrich_scaffold()` method gathers all context needed for generation:

```python
async def enrich_scaffold(self, beat_id, project_id, framework_id) -> SceneScaffold:
```

It queries five data sources in parallel:
1. **SQL**: Beat metadata, project info, voice profile
2. **Neo4j**: Story-layer characters, active setups/payoffs, continuity
3. **pgvector**: Semantic search for relevant research
4. **Framework YAML**: Director config (models, temperature ranges, word counts)
5. **Voice Profile**: POV, tense, anti-patterns, reference sample

## Stage 3: The Tournament

The tournament generates one variant per `WritingStrategy` concurrently via `asyncio.gather()`:

```python
async def generate_scene_variants(self, scaffold, structure_variant) -> list[SceneVariant]:
    tasks = [self._generate_single_variant(scaffold, structure, strategy, i)
             for i, strategy in enumerate(strategies)]
    results = await asyncio.gather(*tasks, return_exceptions=True)
```

### Wildcard (Move-37) Variants

At a configurable ratio (default 0.2), variants are generated in **Wildcard Mode** with higher temperature and looser constraints:

```
WILDCARD MODE (Move-37):
Write freely. Follow the energy, not the outline.
Leave edges unresolved. Don't over-explain.
First thought, best thought.
```

Wildcard variants are **protected from early kill** -- they survive initial quality filtering to ensure creative surprise isn't optimized away.

```python
def _choose_generation_mode(self) -> GenerationMode:
    ratio = self._director_config.wildcard_ratio
    return GenerationMode.WILDCARD if random.random() < ratio else GenerationMode.STANDARD
```

## Stage 4: Scoring

Six criteria with weighted composite scoring:

| Criterion | Weight | Measures |
|-----------|--------|----------|
| `coherence` | 0.20 | Internal logic, beat alignment |
| `voice` | 0.20 | Consistency with target voice |
| `emotional_impact` | 0.15 | Reader engagement, tension |
| `pacing` | 0.15 | Scene rhythm, beat flow |
| `continuity` | 0.15 | Respects prior scenes |
| `payoff_satisfaction` | 0.15 | Delivers on setups |

Optional energy scoring layers (Phase 4B):

- **Canon Energy** (`CanonEnergyScorer`): Measures resonance with the project's existing canon
- **Big Idea** (`BigIdeaScorer`): Alignment with the project's central thesis

When LLM scoring fails, a deterministic MD5-based fallback produces scores in the 40-100 range.

## Tournament Quality Validation

**File:** `backend/app/services/validators/tournament_validator.py`

The `TournamentQualityValidator` runs post-tournament (logging only, does not block):

- **Effect size**: How much did the winner outperform others?
- **P-value**: Statistical significance of the margin
- **Diversity score**: How different were the variants from each other?

This data feeds analytics and parameter tuning.

## Stuck Detection (Phase 4B)

When consecutive tournament rounds fail governance or produce low-diversity output, the system suggests **pivots**:

| Pivot Type | Description |
|------------|-------------|
| `pov_shift` | Switch POV to a different character |
| `time_jump` | Jump forward in time before the scene |
| `dreamlike` | Make the scene oneiric/dreamlike |
| `temperature_spike` | Take bolder creative risks |

Stuck detection uses a `DirectorSession` (persisted to SQL) to track consecutive failures per beat.

## Video Tournament

**File:** `backend/app/services/video_tournament_service.py`

The `VideoTournamentService` follows a simpler pattern -- it records human selections between video draft variants (not automated scoring):

```python
class VideoTournamentService:
    async def get_selection_stats(self, owner_uid) -> SelectionStatsResponse
```

It aggregates model win rates, average scores, and helpfulness metrics. This data drives model routing optimization.

## Empirical Foundation

Tournament parameters are backed by experiments from the Narrative Structure Lab:

| Experiment | Finding | CES Integration |
|------------|---------|-----------------|
| SOT-001 | 3 variants optimal, 40% peak capture | Default variant count |
| SOT-001 | Template contamination at low diversity | Phase 4B Stuck Detection |
| SOT-002 | Wildcards boost Surprise by 15% | Move-37 injection |
| SOT-003 | VoterTrackRecord stabilizes at ~30 rounds | Future Phase 5B |

## Related Documents

- [voice-calibration.md](voice-calibration.md) -- Voice tournaments
- [narrative-governor.md](narrative-governor.md) -- Stage 5 of the pipeline
- [creative-refinement.md](creative-refinement.md) -- Generic refinement loop
