# Adding a Format

> How to create a new creative format using YAML framework definitions.

## Overview

Formats define the creative medium and structure for a project. Each format maps to a **framework** -- a YAML file that specifies beats, interview questions, governor thresholds, and director configuration. CES ships with frameworks for novels, screenplays, comics, memoirs, articles, family videos, TikTok series, newsletters, and audiobooks.

## Architecture

```
Format Selection (UI)
    │
    ▼
formats.yaml ──► format_id
    │
    ▼
FORMAT_TO_FRAMEWORK mapping ──► framework_id
    │
    ▼
FrameworkService.get_framework(framework_id)
    │
    ▼
config/frameworks/{framework_id}.yaml
    │
    ├── beat_vocabulary (story structure)
    ├── interview_pack (questions to ask)
    ├── governor_config (N1-N5 thresholds)
    └── director_config (models, temperatures, word counts)
```

## Step 1: Define the Format

The format catalog is defined in a YAML file that the UI reads for format selection.

Add your format entry:

```yaml
# New format entry
my_format:
  id: my_format
  name: "My Creative Format"
  description: "Short description for the UI"
  medium: prose           # prose | short_form_video | comic | audio
  icon: "pencil"
  category: writing
  source_requirements:
    - voice_samples
    - research_documents
```

## Step 2: Create the Framework YAML

**Directory:** `backend/config/frameworks/`

Create `backend/config/frameworks/my_format.yaml`:

```yaml
id: my_format
name: "My Creative Format"
medium: prose

# Beat vocabulary defines the story structure
beat_vocabulary:
  - name: "opening"
    beat_type: setup
    description: "Establish the world and protagonist"
    target_percentage: 0.10
    act: 1
  - name: "inciting_incident"
    beat_type: catalyst
    description: "The event that disrupts the status quo"
    target_percentage: 0.12
    act: 1
  - name: "rising_action"
    beat_type: escalation
    description: "Complications and obstacles"
    target_percentage: 0.25
    act: 2
  - name: "midpoint"
    beat_type: revelation
    description: "A shift in understanding or stakes"
    target_percentage: 0.08
    act: 2
  - name: "crisis"
    beat_type: escalation
    description: "Maximum tension before resolution"
    target_percentage: 0.20
    act: 3
  - name: "climax"
    beat_type: payoff
    description: "The decisive moment"
    target_percentage: 0.15
    act: 3
  - name: "resolution"
    beat_type: resolution
    description: "New equilibrium established"
    target_percentage: 0.10
    act: 3

# Interview questions for this format
interview_pack:
  - question: "What genre does your story belong to?"
    category: genre
    required: true
  - question: "Who is your protagonist?"
    category: character
    required: true
  - question: "What is the central conflict?"
    category: plot
    required: true

# Governor thresholds for quality enforcement
governor_config:
  n1_threshold: critical
  n2_threshold: critical
  n3_threshold: high
  n4_threshold: high
  n5_threshold: medium
  max_ora_iterations: 5

# Director pipeline configuration
director_config:
  structure_model: "gemini-2.0-pro"
  generation_model: "gemini-2.0-flash"
  scoring_model: "gemini-2.0-flash"
  target_word_count: 2000
  strategies:
    - character_driven
    - plot_driven
    - atmosphere_driven
  temperature_range: [0.5, 0.9]
  wildcard_ratio: 0.2
  wildcard_temperature_range: [0.9, 1.1]
```

### Key Sections

| Section | Purpose |
|---------|---------|
| `beat_vocabulary` | Defines the structural beats (story skeleton) |
| `interview_pack` | Questions to ask the creator during setup |
| `governor_config` | N1-N5 severity thresholds for this format |
| `director_config` | Model selection, word counts, temperature ranges |

### Beat Types

| Type | Meaning |
|------|---------|
| `setup` | Establishes a narrative question |
| `catalyst` | Disrupts the status quo |
| `escalation` | Raises stakes or complications |
| `revelation` | Shifts understanding |
| `payoff` | Resolves a setup |
| `resolution` | Establishes new equilibrium |

## Step 3: Map Format to Framework

The `FORMAT_TO_FRAMEWORK` mapping connects format IDs to framework YAML files.

The `StructureAdapterService` uses this mapping to auto-set `framework_id` on project creation:

```python
# backend/app/services/structure_adapter_service.py

def map_framework(format_id: str) -> str:
    """Map a format_id to its framework_id."""
    return FORMAT_TO_FRAMEWORK.get(format_id, "default")
```

## Step 4: FrameworkService

**File:** `backend/app/services/framework_service.py`

The `FrameworkService` is a read-only YAML loader with in-memory caching:

```python
class FrameworkService:
    def __init__(self, frameworks_dir: Path | None = None):
        self._dir = frameworks_dir or _DEFAULT_FRAMEWORKS_DIR  # config/frameworks/
        self._cache: dict[str, Framework] = {}

    def list_frameworks(self) -> list[FrameworkSummary]
    def get_framework(self, framework_id) -> Framework
    def get_interview_pack(self, framework_id) -> list[InterviewPackQuestion]
    def get_beat_vocabulary(self, framework_id) -> list[BeatDefinition]
    def get_governor_config(self, framework_id) -> GovernorConfig
```

Framework IDs are validated against `^[a-z0-9_-]+$` to prevent path traversal. If a requested framework doesn't exist, the service falls back to `default.yaml`.

## Framework Schema

**File:** `backend/app/schemas/framework.py`

```python
class Framework(BaseModel):
    id: str
    name: str
    medium: str
    beat_vocabulary: list[BeatDefinition] = []
    interview_pack: list[InterviewPackQuestion] = []
    governor_config: GovernorConfig = GovernorConfig()
    director_config: DirectorConfig = DirectorConfig()

class DirectorConfig(BaseModel):
    structure_model: str = "gemini-2.0-pro"
    generation_model: str = "gemini-2.0-flash"
    scoring_model: str = "gemini-2.0-flash"
    target_word_count: int = 2000
    strategies: list[str] = ["character_driven", "plot_driven", "atmosphere_driven"]
    temperature_range: list[float] = [0.5, 0.9]
    wildcard_ratio: float = 0.2
```

## Existing Frameworks

| Framework | Medium | Beats | Special Features |
|-----------|--------|-------|-----------------|
| `default` | generic | 7 | Baseline structure |
| `comic` | comic | 7 | NSL E2-C thresholds, panel-based |
| `article` | prose | 7 | Journalistic structure |
| `memoir` | prose | 7 | Fragmented threads, fabricated_dialogue violation |
| `family_video` | short_form_video | 7 | 2 color variants |

## Format-Specific Behavior

Some formats trigger special processing:

| Format | Special Behavior |
|--------|-----------------|
| Memoir | `StoryBibleService` uses "REAL people" prompts, adds consent fields |
| Comics | `ComicScaffoldService` generates panels, `VisualStyleCanonService` manages style |
| Video | `VideoDirectorService` creates shot plans, routes to video sinks |
| Newsletter | `NewsletterFormatter` structures content for email |

## Testing

Framework YAMLs are tested by loading them through `FrameworkService` and validating the parsed output:

```python
def test_my_format_loads():
    service = FrameworkService()
    fw = service.get_framework("my_format")
    assert fw.id == "my_format"
    assert len(fw.beat_vocabulary) == 7
    assert fw.director_config.target_word_count == 2000
```

## Related Documents

- [learning-engine.md](learning-engine.md) -- Interview packs from frameworks
- [narrative-governor.md](narrative-governor.md) -- Governor config from frameworks
- [society-of-thought.md](society-of-thought.md) -- Director config drives tournament settings
