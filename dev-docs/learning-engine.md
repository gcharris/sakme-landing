# Learning Engine

> How sources become ingredients: interview, extraction, and story bible generation.

## Overview

The Learning Engine transforms raw research materials into structured creative ingredients. It operates in three stages: **Interview** (elicit context from the creator), **Extraction** (pull entities and relationships from uploaded sources), and **Story Bible** (synthesize everything into a canonical reference document).

All three stages run as async services, persist results to PostgreSQL, and optionally project entities into Neo4j for graph queries.

## Architecture

```
                 ┌─────────────────┐
                 │  Upload / Import │
                 └────────┬────────┘
                          ▼
              ┌───────────────────────┐
              │  SourceImportService  │
              │  (parse, chunk, tag)  │
              └───────────┬───────────┘
                          ▼
         ┌────────────────┴────────────────┐
         ▼                                 ▼
┌─────────────────┐              ┌──────────────────┐
│ InterviewService │              │ ExtractionService │
│ (guided Q&A)     │              │ (LLM extraction)  │
└────────┬────────┘              └────────┬─────────┘
         │                                │
         └────────────┬───────────────────┘
                      ▼
            ┌──────────────────┐
            │ StoryBibleService │
            │ (synthesis)       │
            └──────────────────┘
```

## Key Services

### SourceImportService

**File:** `backend/app/services/source_import_service.py`

Handles file upload, NotebookLM ZIP parsing, and content chunking. Detects source categories (characters, world, theme, plot, voice) via tag patterns and distillation headers.

```python
class SourceImportService:
    def __init__(self, db: AsyncSession):
        self.research_service = ResearchService(db)
        self.extraction_service = ExtractionService(db)

    async def import_file(self, *, project_id, filename, content_type, file_data, user_uid):
        """Import a single file: store, extract, and chunk."""

    async def import_notebooklm_zip(self, *, project_id, zip_data, user_uid):
        """Parse NotebookLM export ZIP, detect categories, import each note."""
```

**Category detection** uses two signal layers:

| Signal | Pattern | Example |
|--------|---------|---------|
| Tag patterns | `[CHARACTER]`, `CHAR:` | Explicit creator-applied tags |
| Distillation headers | `**Fatal Flaw**`, `**HARD RULES**` | Headers from NotebookLM distillation prompts |

Tag patterns are defined in `TAG_PATTERNS` and header patterns in `DISTILLATION_HEADERS`, both as compiled regex dictionaries keyed by category name.

### InterviewService

**File:** `backend/app/services/interview_service.py`

Runs a guided interview loop where the system asks questions and records creator responses. Questions are drawn from framework-specific interview packs (YAML-defined) and adapt based on prior answers.

The interview flow uses a **Reverse Prompting** pattern: mid-interview, the engine flips the script and asks "Based on what you've told me, what am I missing?" This surfaces implicit knowledge the creator assumes is obvious.

| Phase | Behavior |
|-------|----------|
| Early | Standard questions (genre, tone, characters) |
| Mid | "Here's what I think I know -- what am I missing?" |
| Late | "Based on your world, here are potential conflicts I see" |

Interview sessions are persisted to PostgreSQL and linked to the project.

### ExtractionService

**File:** `backend/app/services/extraction_service.py`

Runs LLM-powered extraction on uploaded research to pull out structured entities: characters, locations, themes, rules, and plot points. Results are stored as research chunks with embeddings for semantic search.

The extraction pipeline:

1. **Chunking** -- Split source content into overlapping segments
2. **Entity extraction** -- LLM identifies characters, locations, themes per chunk
3. **Deduplication** -- Merge entities that refer to the same thing across chunks
4. **Embedding** -- Generate vector embeddings for semantic search (pgvector)
5. **Graph projection** -- Optionally create Neo4j nodes for extracted entities

### StoryBibleService

**File:** `backend/app/services/story_bible_service.py`

Synthesizes all extracted entities, interview responses, and research into a canonical Story Bible document. The Story Bible is the authoritative reference for a project's creative world.

The service is format-aware. For example, memoir projects get special handling:

- Characters are tagged as "REAL people" in the LLM prompt
- Extra fields: `relationship_to_author`, `living_status`, `consent_status`
- Locations include `time_period`
- Auto-generated privacy narrative rules

```python
async def generate_story_bible(
    self, project_id: uuid.UUID, format_id: str | None = None
) -> StoryBible:
    """Generate a story bible from all available project data."""
```

## Data Flow

1. Creator uploads files or connects NotebookLM
2. `SourceImportService` parses, categorizes, and chunks content
3. `ExtractionService` runs LLM extraction on chunks to find entities
4. Entities are stored in PostgreSQL and projected to Neo4j
5. `InterviewService` asks targeted questions based on what's been found
6. `StoryBibleService` synthesizes everything into the canonical bible
7. The Story Bible feeds into Voice Calibration and Scene Generation

## Database Models

| Model | Table | Purpose |
|-------|-------|---------|
| `ResearchSource` | `research_sources` | Uploaded file metadata |
| `ResearchChunk` | `research_chunks` | Chunked content with embeddings |
| `InterviewSession` | `interview_sessions` | Interview state and responses |
| `StoryBible` | `story_bibles` | Generated story bible content |
| `CanonWeightHint` | `canon_weight_hints` | Source importance signals |

## API Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/api/v1/projects/{id}/research` | Upload research file |
| POST | `/api/v1/source/import` | Import file with extraction |
| POST | `/api/v1/source/import/notebooklm` | Import NotebookLM ZIP |
| GET | `/api/v1/interview/{project_id}` | Get interview state |
| POST | `/api/v1/interview/{project_id}/answer` | Submit interview answer |
| POST | `/api/v1/projects/{id}/story-bible/generate` | Generate story bible |

## Related Documents

- [knowledge-graph.md](knowledge-graph.md) -- Entity storage in Neo4j
- [voice-calibration.md](voice-calibration.md) -- Next step after story bible
- [adding-a-source.md](adding-a-source.md) -- How to add new source connectors
