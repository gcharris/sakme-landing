# Testing

> CES has 388 backend test files with ~5,265 test functions and 60 frontend test files. Tests are async-native, use dependency injection for isolation, and mock all external services (LLM, Neo4j, Firebase) for deterministic, fast execution. The backend runs in CI via GitHub Actions; frontend CI is blocked by a pre-existing ESM configuration issue.

## Key Files

| File | Purpose |
|------|---------|
| `backend/tests/conftest.py` | Main test fixtures (database, client, auth) |
| `backend/tests/integration/conftest.py` | Integration test markers and Neo4j availability |
| `backend/app/services/llm_client.py` | `MockLLMClient` — canned response LLM |
| `backend/pyproject.toml` | pytest configuration (`asyncio_mode = "auto"`) |
| `.github/workflows/ci.yml` | CI pipeline (lint + test + failure notification) |
| `frontend/vite.config.ts` | Vitest configuration |
| `frontend/src/setupTests.ts` | Frontend test setup |

## Backend Testing

### Test Organization

```
backend/tests/
├── conftest.py              # Main fixtures (100 lines)
├── services/                # 202 files — Business logic (LARGEST)
├── api/                     # 63 files — REST endpoints
├── schemas/                 # 17 files — Pydantic validation
├── integration/             # 15 files — E2E pipeline tests
├── models/                  # 12 files — ORM model tests
├── unit/                    # 10 files — Misc unit tests
├── core/                    # 8 files — Config, database, security
├── architect/               # 3 files — Agent tests
├── graph/                   # 3 files — Neo4j tests
├── infra/                   # 3 files — Infrastructure
├── middleware/              # 2 files — Middleware tests
├── events/                  # 2 files — Event sourcing
├── migrations/              # 2 files — Alembic tests
└── security/                # 1 file — Auth tests
```

### Statistics

| Metric | Value |
|--------|-------|
| Test files | 388 |
| Test functions | ~5,265 |
| Test classes | 257 |
| Async tests | ~2,811 |
| CI run time | ~30-60 seconds |

### Core Fixtures (`conftest.py`)

**Database:** In-memory SQLite with aiosqlite (not PostgreSQL). Tables are created before each test and dropped after.

```python
# In-memory database for speed
TEST_DATABASE_URL = "sqlite+aiosqlite:///file::memory:?cache=shared&uri=true"

@pytest.fixture(autouse=True)
async def setup_database():
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with test_engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
```

**HTTP Client:** AsyncClient (httpx) with ASGI transport — no network calls.

```python
@pytest.fixture
async def client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

**Auth Override:** All endpoints see a hardcoded test user.

```python
TEST_USER = {"uid": "test-user-001", "email": "test@example.com", "name": "Test User"}

app.dependency_overrides[get_current_user] = lambda: TEST_USER
app.dependency_overrides[get_db] = override_get_db
app.dependency_overrides[get_graph_service] = get_mock_graph_service()
```

**Environment:** Tests run with `CES_ALLOW_DEV_AUTH=true` and `CES_USE_MOCK_LLM=true` set at import time.

### MockLLMClient

**File:** `backend/app/services/llm_client.py`

A deterministic LLM mock that returns canned responses based on keyword matching in the prompt.

```python
llm = MockLLMClient(canned_responses={
    "analyze sentiment": '{"sentiment": "positive", "score": 0.85}',
    "extract entities": '[{"type": "person", "value": "John"}]',
})

response = await llm.generate(
    user_prompt="Analyze sentiment: I love this!",
)
# Returns: LLMResponse with content matching "analyze sentiment" key

assert llm.call_count == 1
assert "Analyze sentiment" in llm.call_log[0]["user_prompt"]
```

Features:
- Keyword-based response matching (case-insensitive)
- `call_count` and `call_log` for verification
- Token usage tracking (`TokenUsage` dataclass)
- Inherits from `LLMClient` base class (polymorphic substitution)

### Testing Patterns

#### Pattern 1: Service Tests (202 files)

Services are tested via constructor injection. Dependencies are mocked with `MagicMock`/`AsyncMock`.

```python
@pytest.fixture
def mock_embedding_service():
    service = MagicMock()
    service.embed_text = AsyncMock(return_value=[0.5] * 768)
    return service

@pytest.fixture
def detector(mock_embedding_service, mock_db_session):
    return ContradictionDetector(
        embedding_service=mock_embedding_service,
        db=mock_db_session,
    )

class TestContradictionDetector:
    @pytest.mark.asyncio
    async def test_detects_contradiction(self, detector):
        result = await detector.detect(entity_id=..., flash_claim=...)
        assert result.status == ContradictionStatus.CONTRADICTED
```

#### Pattern 2: API Endpoint Tests (63 files)

Endpoints are tested via AsyncClient with dependency overrides.

```python
@pytest.mark.asyncio
async def test_create_project(self, client):
    response = await client.post(
        "/api/v1/projects",
        json={"title": "My Novel", "format": "novel"},
    )
    assert response.status_code == 201
    assert response.json()["title"] == "My Novel"
```

Error path testing covers `400`, `404`, `409`, `500` status codes.

#### Pattern 3: Database Transaction Tests

Direct database access for service-level testing.

```python
@pytest.mark.asyncio
async def test_create_codex_chunk(self, db_session):
    chunk = CodexChunk(
        tenant_id=uuid.uuid4(),
        content="Knowledge...",
        source_type="interview",
    )
    db_session.add(chunk)
    await db_session.commit()

    result = await db_session.execute(
        select(CodexChunk).where(CodexChunk.content == "Knowledge...")
    )
    assert result.scalar() is not None
```

#### Pattern 4: Async Testing

All async functions are automatically discovered by pytest-asyncio (`asyncio_mode = "auto"`).

```python
# No decorator needed — auto-discovery
async def test_concurrent_operations(billing):
    results = await asyncio.gather(
        billing.create_checkout_session(...),
        billing.create_checkout_session(...),
    )
    assert len(results) == 2
```

### Integration Tests

**File:** `backend/tests/integration/conftest.py`

Integration tests can optionally use real Neo4j if available:

```python
@pytest.fixture
def neo4j_available():
    uri = os.environ.get("CES_NEO4J_URI")
    if not uri:
        pytest.skip("Neo4j not available")
```

15 integration test files cover full pipeline flows (onboarding → brainstorm → generation).

## Frontend Testing

### Framework: Vitest

**Configuration** (`frontend/vite.config.ts`):

```typescript
test: {
    environment: 'node',
    globals: true,
    setupFiles: ['./src/setupTests.ts'],
    pool: 'forks',       // Per-test isolation
    singleFork: true,
}
```

**Libraries:**

| Library | Purpose |
|---------|---------|
| `vitest` | Test runner (v4.0.18) |
| `@testing-library/react` | Component rendering and queries |
| `@testing-library/jest-dom` | DOM assertion matchers |
| `jsdom` | DOM simulation |

### Frontend Test Organization (60 files)

```
frontend/
├── __tests__/onboarding/          # Onboarding component tests
├── __tests__/voice/               # Voice calibration tests
├── src/api/__tests__/             # API client tests
├── src/stores/__tests__/          # State management tests
├── src/pages/__tests__/           # Page-level tests (13+ files)
└── src/__tests__/                 # General component tests
```

### Frontend Test Patterns

```typescript
vi.mock('../../src/lib/api', () => ({
  apiFetch: vi.fn(),
}));

describe('SecondBrainStep', () => {
  const mockApiFetch = vi.mocked(apiFetch);

  beforeEach(() => {
    mockApiFetch.mockResolvedValue(new Response('{}', { status: 200 }));
  });

  it('renders all 4 connector cards', () => {
    render(<SecondBrainStep {...defaultProps} />);
    expect(screen.getByText('Readwise')).toBeInTheDocument();
  });
});
```

## CI/CD Pipeline

**File:** `.github/workflows/ci.yml`

### Three Jobs

**1. Lint** (Ubuntu, Python 3.12)

```bash
ruff format --check        # Formatting
ruff check app/ tests/     # Linting
# Custom check: prevent direct LLM imports (must use BudgetOrchestrator)
```

**2. Test** (Ubuntu, Python 3.12)

```bash
pip install -e ".[dev]" && pip install aiosqlite
pytest tests/ -v --tb=short
```

**3. Notify Failure** (conditional)

On lint or test failure, sends an HMAC-signed webhook to `CI_WEBHOOK_URL` with run ID, branch, SHA, and logs URL. This enables the two-AI fix pipeline (Observation Service picks up the failure).

### Triggers

- Push to `main`
- Pull requests to `main`

### Known CI Issues

- Frontend tests (`vitest`) are **not** run in CI due to a pre-existing ESM configuration issue
- The issue is documented but not yet resolved

## Running Tests Locally

### Backend

```bash
cd backend
pip install -e ".[dev]"
pip install aiosqlite

# Run all tests
pytest tests/ -v

# Run specific domain
pytest tests/services/test_billing_service.py -v

# Run with coverage
pytest tests/ --cov=app --cov-report=html
```

### Frontend

```bash
cd frontend
npm install
npx vitest run
```

## Design Principles

| Principle | Implementation |
|-----------|---------------|
| **Isolation** | In-memory SQLite, AsyncClient (no network), MockLLMClient |
| **Determinism** | Keyword-matching canned responses, consistent fixtures |
| **Async-native** | All tests use `async/await`, pytest-asyncio auto-discovery |
| **Speed** | ~30-60 seconds for full backend suite (5,265 tests) |
| **Dependency injection** | Services accept mocks via constructor; endpoints via FastAPI overrides |

## Known Limitations

1. **No real LLM tests** — All tests use `MockLLMClient`. Real Vertex AI integration tests are needed but not yet implemented.
2. **Pre-existing failures** — ~100 failures in `telegram_sink`, `tournament`, `youtube_rss` test files predate recent work.
3. **No Neo4j locally** — Graph-dependent tests skip unless `CES_NEO4J_URI` is set.
4. **Frontend CI blocked** — Vitest runs locally but not in CI due to ESM issue.
5. **No coverage reporting** — pytest-cov is installed but not enforced in CI.

## Related Documents

- [Architecture Overview](architecture-overview.md) — System architecture
- [Data Model](data-model.md) — Models under test
