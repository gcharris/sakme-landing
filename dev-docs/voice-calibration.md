# Voice Calibration

> Tournament-based voice learning, temperature strategies, and voice profiles.

## Overview

Voice Calibration learns a creator's writing voice by running **mini-tournaments** -- generating multiple variants with different strategies and temperatures, then having the creator pick their preferred style. The winning characteristics (POV, tense, metaphor domains, anti-patterns) are captured in a `VoiceProfile` that guides all subsequent generation.

## Architecture

```
┌──────────────────────────────────────────┐
│         Voice Calibration Flow            │
│                                           │
│  Reference Samples                        │
│        ▼                                  │
│  Exemplar Picker (select best samples)    │
│        ▼                                  │
│  Tournament (3 strategies x temperatures) │
│        ▼                                  │
│  Creator Selection (human picks winner)   │
│        ▼                                  │
│  VoiceProfile (persisted to SQL)          │
│        ▼                                  │
│  VoiceBundle (used in generation)         │
└──────────────────────────────────────────┘
```

## Key Service

**File:** `backend/app/services/voice_calibration_service.py`

### VoiceCalibrationService

The main service that orchestrates voice learning:

```python
class VoiceCalibrationService:
    async def start_calibration(self, project_id, reference_samples) -> CalibrationSession
    async def calibrate_ensemble(self, project_id, characters) -> dict[UUID, VoiceBundle]
    async def run_tournament(self, session_id, round_number) -> list[VoiceVariant]
    async def record_selection(self, session_id, variant_id) -> VoiceProfile
```

### calibrate_ensemble()

For multi-character projects, runs mini-tournaments per character concurrently via `asyncio.gather()`. Each character gets its own `VoiceBundle` based on its description and role.

### Tournament Mechanics

Each tournament round generates variants across `WritingStrategy` values:

| Strategy | Focus | Temperature Range |
|----------|-------|-------------------|
| `character_driven` | Deep POV, internal monologue | 0.5 -- 0.9 |
| `plot_driven` | Action, pacing, tension | 0.5 -- 0.9 |
| `atmosphere_driven` | Setting, mood, sensory detail | 0.5 -- 0.9 |

Temperature is randomized within the range for each variant, creating natural diversity. This follows the empirically validated parameters from SOT-001 experiments.

## VoiceProfile Model

**File:** `backend/app/models/voice_profile.py`

The `VoiceProfile` captures learned voice characteristics:

| Field | Type | Purpose |
|-------|------|---------|
| `pov` | String | Point of view (first, third_limited, third_omniscient) |
| `tense` | String | Narrative tense (past, present) |
| `voice_type` | String | Overall style label |
| `metaphor_domains` | JSON | Preferred metaphor categories |
| `anti_patterns` | JSON | Phrases/patterns to avoid |
| `reference_sample` | Text | Best reference text for drift detection |
| `sentence_rhythm` | String | Short/long sentence preferences |
| `vocabulary_level` | String | Simple, moderate, literary |
| `characteristic_phrases` | JSON | Signature expressions |

## VoiceBundle Schema

**File:** `backend/app/schemas/voice.py`

The `VoiceBundle` is a Pydantic model passed to generation services. It's built from the stored `VoiceProfile`:

```python
@dataclass
class VoiceBundle:
    pov: str
    tense: str
    voice_type: str
    metaphor_domains: list[str]
    sentence_rhythm: str | None
    vocabulary_level: str | None
    characteristic_phrases: list[str]
    anti_patterns: list[str]
    reference_sample: str | None
```

## Voice Drift Detection

During scene generation, the `DirectorService` computes **voice drift** by comparing the cosine distance between a generated variant's embedding and the reference sample embedding:

```python
async def _compute_voice_drift(self, content, ref_embedding) -> float:
    """Cosine distance between content and reference sample. 0.0 = identical."""
```

High drift scores (closer to 1.0) indicate the generated text has wandered from the creator's established voice.

## Frontend Components

The voice calibration UI was ported from a Svelte prototype to React:

| Component | File | Purpose |
|-----------|------|---------|
| `VoiceCalibration.tsx` | `frontend/src/components/` | Main calibration flow |
| `TournamentLauncher.tsx` | `frontend/src/components/` | Start tournament UI |
| `VariantCard.tsx` | `frontend/src/components/` | Display one variant for selection |
| `ExemplarPicker.tsx` | `frontend/src/components/` | Select reference samples |
| `ExemplarPreview.tsx` | `frontend/src/components/` | Preview exemplar content |
| `VoiceProvider` | `frontend/src/contexts/` | React context for voice state |

## API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/voice-calibration/{project_id}/start` | Begin calibration |
| POST | `/api/v1/voice-calibration/{project_id}/tournament` | Run tournament round |
| POST | `/api/v1/voice-calibration/{project_id}/select` | Record creator selection |
| GET | `/api/v1/voice-calibration/{project_id}/profile` | Get current voice profile |

## Empirical Validation

Voice calibration parameters are backed by empirical research from the Narrative Structure Lab:

- **SOT-001**: Tournament wins 40% peak capture rate vs single-shot. 3 variants is optimal.
- **SOT-002**: Wildcard injection at 0.10--0.20 ratio prevents template recycling, +15% surprise.
- **Temperature range [0.7, 0.9]**: Balances creativity with coherence.

## Related Documents

- [society-of-thought.md](society-of-thought.md) -- Tournament architecture used here
- [creative-refinement.md](creative-refinement.md) -- Post-generation quality loop
- [narrative-governor.md](narrative-governor.md) -- N4 voice drift detection
