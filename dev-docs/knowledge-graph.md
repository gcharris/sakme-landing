# Knowledge Graph

> Neo4j integration, GraphRAG, entity relationships, and graph layers.

## Overview

CES uses Neo4j as a **projection layer** on top of PostgreSQL. SQL is the source of truth for all data; Neo4j stores entity relationships that enable graph queries such as "which characters appear together?", "what setups lack payoffs?", and "how are themes connected to scenes?".

All graph operations are **project-scoped** via a `project_uid` property on every node. The graph has two layers: **research** (raw extracted entities) and **story** (promoted entities actively used in generation).

## Architecture

```
┌──────────────────────────────────────────────────┐
│                   Neo4j Graph                     │
│                                                   │
│  Research Layer        Story Layer                │
│  ┌─────────┐          ┌─────────┐                │
│  │Character│──PROMOTED_FROM──│Character│          │
│  │Location │          │Location │                 │
│  │Theme    │          │Theme    │                 │
│  │Beat     │          │Beat     │                 │
│  └─────────┘          │Scene    │                 │
│                       │Setup    │                 │
│                       └─────────┘                │
│                                                   │
│  Flash Tier (CES 2.0)                            │
│  ┌─────────┐                                     │
│  │Entity   │──SOURCED_FROM──│Notebook│           │
│  └─────────┘                └────────┘           │
└──────────────────────────────────────────────────┘
```

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/graph/graph_service.py` | CRUD operations on the knowledge graph |
| `backend/app/graph/neo4j_client.py` | Async Neo4j driver, tenant isolation, Cypher injection prevention |
| `backend/app/graph/schema.py` | Schema initialization (constraints, indexes) |

## Node Labels

Defined in `ALLOWED_LABELS` in `backend/app/graph/graph_service.py`:

| Label | Layer | Purpose |
|-------|-------|---------|
| `Character` | research/story | Named characters with role, description |
| `Location` | research/story | Settings and places |
| `Theme` | research/story | Thematic elements |
| `Beat` | research/story | Story beats with type (setup/payoff) |
| `NarrativeRule` | research/story | World rules and constraints |
| `Scene` | story | Generated scenes |
| `Setup` | story | Narrative setups (open questions) |
| `Project` | -- | Root node per project |
| `Entity` | flash | CES 2.0 Flash tier entities |
| `Notebook` | flash | CES 2.0 NotebookLM source |

## Relationship Types

Defined in `ALLOWED_RELATIONSHIPS`:

| Relationship | From | To | Purpose |
|-------------|------|-----|---------|
| `PROMOTED_FROM` | Story node | Research node | Tracks which research entities became story canon |
| `FOLLOWS` | Scene | Scene | Scene ordering (by `sequence_order`) |
| `APPEARS_IN` | Character | Scene | Character presence in scenes |
| `APPLIES_TO` | NarrativeRule | Scene | Rule enforcement scope |
| `SETS_UP` | Beat | Beat | Setup-payoff chains |
| `PAYS_OFF` | Beat | Beat | Payoff links |
| `EXPLORED_IN` | Theme | Scene | Theme usage |
| `BELONGS_TO` | Any | Project | Project scoping |
| `INTRODUCED_IN` | Setup | Scene | Where a narrative question was introduced |
| `SOURCED_FROM` | Entity | Notebook | Flash tier provenance |
| `RELATES_TO` | Entity | Entity | Flash tier entity connections |

## GraphService API

**File:** `backend/app/graph/graph_service.py`

The `GraphService` class provides project-scoped CRUD operations:

```python
class GraphService:
    async def create_node(self, label, project_uid, properties, graph_layer="research")
    async def get_node(self, label, node_uid)
    async def update_node(self, label, node_uid, properties)
    async def delete_node(self, label, node_uid)
    async def list_nodes(self, label, project_uid, *, graph_layer=None, limit=100)
    async def promote_node(self, label, source_uid, project_uid, properties=None)
    async def create_relationship(self, from_label, from_uid, rel_type, to_label, to_uid, ...)
    async def get_relationships(self, node_uid, *, direction="both", rel_type=None)
    async def execute_query(self, query, **params)
    async def init_project_node(self, project_uid, title)
    async def delete_project_graph(self, project_uid)
    async def get_project_stats(self, project_uid)
```

### Node Promotion (Research to Story)

The `promote_node()` method handles the research-to-story transition:

1. MERGEs the source node if missing (SQL is source of truth)
2. Creates a new story-layer node with copied properties
3. Creates a `PROMOTED_FROM` relationship back to the research node
4. Applies any property overrides

### Security

All label and relationship type parameters are validated against allowlists (`ALLOWED_LABELS`, `ALLOWED_RELATIONSHIPS`). Property keys are validated against `^[a-zA-Z_][a-zA-Z0-9_]*$` to prevent Cypher injection.

## Tenant Isolation

**File:** `backend/app/graph/neo4j_client.py`

The `TenantScopedSession` class wraps Neo4j sessions to automatically inject tenant filtering:

```python
class TenantScopedSession:
    async def run(self, query, parameters=None)   # Tenant-filtered
    async def run_raw(self, query, parameters=None) # No filtering (admin only)
```

The `inject_tenant_filter()` function modifies Cypher queries:
- `MATCH` queries: Adds `Tenant` node match with `OWNS` relationship check
- `CREATE` queries: Links new nodes to the tenant via `OWNS`
- `MERGE` queries: Ensures tenant context

Tenant IDs are validated against Cypher injection patterns (`DETACH DELETE`, `DROP`, `CALL db.`, comments).

## Schema Initialization

**File:** `backend/app/graph/schema.py`

Called on application startup via `init_graph_schema()`. Creates uniqueness constraints on `uid` for all node labels and indexes on `project_uid` for efficient project-scoped queries.

All statements use `IF NOT EXISTS`, making initialization idempotent.

## Graph Layers

| Layer | Purpose | Populated by |
|-------|---------|-------------|
| `research` | Raw extracted entities, not yet curated | ExtractionService, SourceImportService |
| `story` | Curated entities used in generation | promote_node(), DirectorService |
| `flash` | Pre-processed NotebookLM synthesis | FlashSeedingService (CES 2.0) |

## Current Limitations

1. **Neo4j not always running** -- The application starts gracefully without Neo4j. Graph operations return empty results.
2. **Non-atomic triple-write** -- SQL, Neo4j, and EventStore writes are not transactional. SQL is authoritative; graph can be reprojected.
3. **No full GraphRAG** -- Semantic search uses pgvector in PostgreSQL. Neo4j is used for structural queries only.

## Related Documents

- [data-model.md](data-model.md) -- SQL models that feed the graph
- [learning-engine.md](learning-engine.md) -- Services that populate the graph
- [narrative-governor.md](narrative-governor.md) -- Uses graph for kill switch checks
