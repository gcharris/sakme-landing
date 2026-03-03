# Adding a Source

> How to add a new input connector for content ingestion.

## Overview

Sources are input connectors that bring external content into the Sovereign Codex. CES supports several source types: file uploads, NotebookLM exports, Twitter, Gmail, YouTube, Pinterest, and PKMS tools (Roam, Obsidian). Adding a new source involves creating an ingestion service, OAuth flow (if needed), and Codex integration.

## Existing Sources

| Source | Service | OAuth | Status |
|--------|---------|-------|--------|
| File Upload | `SourceImportService` | No | Complete |
| NotebookLM ZIP | `SourceImportService` | No | Complete |
| Twitter | `SourceIngestionService` | Yes | Complete |
| Gmail | `SourceIngestionService` | Yes | Complete |
| YouTube | `YouTubeSourceService` | Yes | Complete |
| Pinterest | `auth_providers.py` | Yes | OAuth only |
| Roam Research | Planned | No (JSON export) | Spec complete |
| Obsidian | Planned | No (vault folder) | Spec complete |

## Source Architecture

```
External Platform ──► OAuth / File Upload
                          │
                          ▼
                   SourceConnection (model)
                          │
                          ▼
                   Ingestion Service
                   (fetch + transform)
                          │
                          ▼
                   CodexChunk (EXTERNAL tier)
                          │
                          ▼
                   Crystallization (on approval)
                          │
                          ▼
                   CodexChunk (PROJECT/SOVEREIGN tier)
```

## Step 1: Add SourceConnection Support

Sources that require OAuth use the `SourceConnection` model to store connection state:

```python
# backend/app/models/source_connection.py

class SourceConnection(Base, UUIDMixin, TimestampMixin):
    tenant_id: UUID
    source_type: str           # "twitter", "gmail", "my_source"
    status: ConnectionStatus   # CONNECTED, EXPIRED, REVOKED
    external_user_id: str | None
    display_name: str | None
    metadata_json: dict        # Platform-specific data
```

Access tokens are stored encrypted in `VaultEntry`:

```python
class VaultEntry(Base, UUIDMixin):
    tenant_id: UUID
    key_name: str             # "my_source_access_token"
    encrypted_value: str      # Fernet-encrypted
```

## Step 2: Create Ingestion Service

**File:** `backend/app/services/my_source_ingestion_service.py`

Follow the pattern from `backend/app/services/source_ingestion_service.py`:

```python
from app.core.vault_crypto import VaultCrypto
from app.models.codex import CodexChunk, TrustTier
from app.models.source_connection import SourceConnection
from app.schemas.ingestion import IngestionResult

class MySourceIngestionService:
    def __init__(self, db: AsyncSession, vault: VaultCrypto):
        self.db = db
        self.vault = vault

    async def ingest(
        self,
        connection_id: uuid.UUID,
        since: str | None = None,
        max_items: int = 100,
    ) -> IngestionResult:
        # 1. Load connection and decrypt access token
        conn, access_token = await self._get_connection_and_token(connection_id)

        # 2. Fetch content from platform API
        items = await self._fetch_items(access_token, since, max_items)

        # 3. Create CodexChunks for each item
        created, skipped = 0, 0
        for item in items:
            content_hash = hashlib.sha256(item.content.encode()).hexdigest()

            # Dedup check
            existing = await self._find_existing(conn.tenant_id, content_hash)
            if existing:
                skipped += 1
                continue

            chunk = CodexChunk(
                tenant_id=conn.tenant_id,
                content=item.content,
                content_hash=content_hash,
                source_type="my_source",
                source_uri=item.url,
                trust_tier=TrustTier.EXTERNAL,
                director_blessing_score=0.0,
                voice_dna_ready=False,
            )
            self.db.add(chunk)
            created += 1

        await self.db.flush()
        return IngestionResult(created=created, skipped=skipped, total=len(items))
```

Key patterns:
- **Decrypt tokens via VaultCrypto**: Never store raw tokens in memory longer than needed
- **Dedup via content_hash**: SHA-256 hash prevents duplicate chunks
- **EXTERNAL trust tier**: All ingested content starts as untrusted
- **blessing_score=0.0**: Must be crystallized (approved) to become trusted

## Step 3: Add OAuth Flow (If Needed)

**File:** `backend/app/api/v1/auth_providers.py`

Follow the existing Google/Twitter/Pinterest OAuth patterns:

```python
@router.get("/my_source/initiate")
async def initiate_my_source_oauth(
    source: str = Query(default="web"),
    db: AsyncSession = Depends(get_db),
    user=Depends(get_current_user),
):
    """Generate OAuth URL for MySource."""
    state = generate_oauth_state(user.tenant_id, source)
    auth_url = f"https://my-source.com/oauth/authorize?client_id={settings.my_source_client_id}&state={state}"
    return {"authorization_url": auth_url}

@router.get("/my_source/callback")
async def my_source_callback(
    code: str, state: str,
    db: AsyncSession = Depends(get_db),
):
    """Handle OAuth callback, store tokens."""
    tenant_id = validate_oauth_state(state)
    tokens = await exchange_code_for_tokens(code)

    # Store encrypted access token
    vault = VaultCrypto()
    encrypted = vault.encrypt(tokens["access_token"])
    # Create VaultEntry and SourceConnection...
```

## Step 4: Add Configuration

**File:** `backend/app/core/config.py`

```python
class Settings(BaseSettings):
    my_source_client_id: str = ""
    my_source_client_secret: str = ""
    my_source_callback_url: str = "https://ces-backend-331630083873.us-central1.run.app/api/v1/connections/my_source/callback"
```

## Step 5: Wire Into Source Discovery

If your source should appear in the onboarding flow, update the source discovery service to detect connected instances:

**File:** `backend/app/services/source_discovery_service.py`

```python
# Add to source type detection
SOURCE_CONFIGS = {
    # ... existing entries ...
    "my_source": SourceConfig(
        display_name="MySource",
        icon="my_source",
        requires_oauth=True,
        categories=["semantic"],
    ),
}
```

## Step 6: Add Tests

```python
# backend/tests/services/test_my_source_ingestion.py

@pytest.mark.asyncio
async def test_ingest_creates_codex_chunks(db_session):
    vault = VaultCrypto(key=Fernet.generate_key().decode())
    service = MySourceIngestionService(db=db_session, vault=vault)
    result = await service.ingest(connection_id=conn.id)
    assert result.created > 0

@pytest.mark.asyncio
async def test_dedup_skips_existing(db_session):
    # Ingest same content twice
    service = MySourceIngestionService(db=db_session, vault=vault)
    result1 = await service.ingest(connection_id=conn.id)
    result2 = await service.ingest(connection_id=conn.id)
    assert result2.skipped == result1.created
```

## Source Categories

Sources are classified into categories that determine how they're used:

| Category | Examples | Usage |
|----------|----------|-------|
| Semantic | Twitter, Gmail, Roam | Text content for extraction and voice |
| Structural | Roam, Obsidian | Graph structure from links/backlinks |
| Audio | Podcast uploads | Voice cloning via KitsAI |
| Visual | Pinterest, Instagram | Aesthetic profiling |
| Social | YouTube, TikTok | Engagement data |

Not all sources can be flattened through text analysis. Visual and audio sources require specialized processing pipelines.

## Related Documents

- [sovereign-codex.md](sovereign-codex.md) -- How ingested content becomes trusted knowledge
- [auth-system.md](auth-system.md) -- OAuth provider setup
- [byok-model.md](byok-model.md) -- VaultCrypto for token encryption
