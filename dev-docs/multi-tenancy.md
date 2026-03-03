# Multi-Tenancy

> CES isolates tenants at two levels: PostgreSQL schema-per-tenant (via `SET search_path`) and Neo4j Cypher-level filtering (via `inject_tenant_filter()`). Every request carries a tenant identity through middleware, contextvars, and request state.

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/middleware/tenant_context.py` | Extract `X-Tenant-ID`, validate, store in contextvars |
| `backend/app/core/database.py` | Schema-per-tenant sessions (`get_db_with_tenant`) |
| `backend/app/core/tenant_resolver.py` | Email-to-tenant lookup |
| `backend/app/models/tenant.py` | Tenant model, roles, tiers, status |
| `backend/app/graph/neo4j_client.py` | Cypher-level tenant filtering |

## How Tenant Identity Flows

```
Request arrives
    │
    ▼
┌───────────────────────────────┐
│ TenantContextMiddleware       │
│ Extract X-Tenant-ID header    │
│ Validate format (regex)       │
│ Check for injection patterns  │
│ Store in contextvars + state  │
└───────────┬───────────────────┘
            │
    ┌───────┴───────┐
    ▼               ▼
┌────────────┐  ┌────────────────┐
│ PostgreSQL │  │     Neo4j      │
│ SET search │  │ inject_tenant  │
│ _path to   │  │ _filter() on   │
│ tenant_{id}│  │ all Cypher     │
└────────────┘  └────────────────┘
```

## Tenant Context Middleware

**File:** `backend/app/middleware/tenant_context.py`

The middleware runs on every request and:

1. Extracts the `X-Tenant-ID` header
2. Validates the format against `^[a-zA-Z0-9][a-zA-Z0-9_-]{0,126}[a-zA-Z0-9]?$`
3. Checks for injection patterns (SQL, path traversal, HTML, backtick, variable injection)
4. Stores the tenant_id in Python `contextvars` (async-safe thread-local)
5. Stores the tenant_id in `request.state.tenant_id`
6. Echoes the tenant_id back in the response `X-Tenant-ID` header
7. Resets the context after the request completes

### Tenant ID Constraints

- Alphanumeric characters, hyphens, and underscores only
- 1-128 characters
- Rejected patterns: `;`, `'`, `"`, `--`, `/`, `\`, `<`, `>`, `$`, `` ` ``, whitespace

### Accessing Tenant ID in Code

```python
from app.middleware.tenant_context import get_current_tenant_id, require_tenant

# Returns str | None (no error if missing)
tenant_id = get_current_tenant_id()

# Raises TenantRequiredError if missing
tenant_id = require_tenant()
```

## PostgreSQL: Schema-Per-Tenant

**File:** `backend/app/core/database.py`

Each tenant gets an isolated PostgreSQL schema named `tenant_{sanitized_id}`. The sanitization removes non-alphanumeric characters except underscores to prevent SQL injection.

### Two Session Types

| Dependency | Schema | Use Case |
|-----------|--------|----------|
| `get_db()` | `public` | Platform-level operations (user lookup, tenant provisioning) |
| `get_db_with_tenant()` | `tenant_{id}` | User-scoped operations (projects, scenes, voice profiles) |

### How `get_db_with_tenant()` Works

1. Reads `X-Tenant-ID` from middleware
2. Sanitizes the tenant ID
3. Executes `SET search_path TO tenant_{id}, public`
4. Yields the session for the request
5. On cleanup: resets `SET search_path TO public` before returning connection to pool

### Schema Management

```python
# Provision a new tenant schema
await create_tenant_schema(tenant_id)

# Check if schema exists
exists = await tenant_schema_exists(tenant_id)

# Drop tenant schema (destructive)
await drop_tenant_schema(tenant_id, cascade=True)
```

## Neo4j: Cypher-Level Filtering

**File:** `backend/app/graph/neo4j_client.py`

Neo4j does not have native multi-tenancy. CES implements tenant isolation by modifying Cypher queries at runtime.

### `inject_tenant_filter()`

Modifies Cypher queries to scope all operations to the current tenant:

| Query Type | Modification |
|-----------|-------------|
| **MATCH** | Inserts tenant match at beginning + adds `WHERE EXISTS((t)-[:OWNS]-(...))` |
| **CREATE** | Wraps in tenant MATCH + establishes `OWNS` relationship |
| **MERGE** | Adds tenant MATCH prefix |

### Query Validation

Before injection, queries are validated to prevent dangerous operations:

- Blocks `DETACH DELETE`
- Blocks `DROP`
- Blocks `CALL db.*`
- Blocks SQL-style comments (`--`)

### Tenant Node Management

```python
# Ensure tenant node exists in graph
await ensure_tenant_node(tenant_id)
```

Creates a `(:Tenant {tenant_id: "..."})` node if it doesn't already exist.

## Tenant Model

**File:** `backend/app/models/tenant.py`

### Core Fields

| Field | Type | Purpose |
|-------|------|---------|
| `tenant_id` | str (max 128) | Unique identifier |
| `display_name` | str, nullable | Human-readable name |
| `persona_name` | str, default "Muse" | AI assistant name for this tenant |
| `admin_email` | str, nullable | Primary contact email |
| `schema_name` | str | PostgreSQL schema name |
| `neo4j_node_id` | str, nullable | Graph database node reference |
| `role` | TenantRole | Access level |
| `tier` | TenantTier | Subscription level (gates features) |
| `status` | TenantStatus | Provisioning state |

### Roles

| Role | Description |
|------|-------------|
| `DIRECTOR` | Sovereign owner (the Director/admin) |
| `ARCHITECT` | AI builder agent |
| `AGENT` | Other AI agents |
| `OPERATOR` | Human operators (default for new users) |
| `GUEST` | Read-only access |

### Tiers

| Tier | Description |
|------|-------------|
| `FREE` | Locked at defaults, Telegram-only |
| `PRO` | Full Mixing Board access, web + Telegram |
| `SOVEREIGN` | All features + BUILD mode + Architect access |

### Subscription Fields

| Field | Type | Purpose |
|-------|------|---------|
| `subscription_tier` | str | explorer, creator, or showrunner |
| `billing_customer_id` | str, nullable | Provider customer ID |
| `billing_subscription_id` | str, nullable | Provider subscription ID |
| `subscription_status` | str | active or inactive |
| `subscription_ends_at` | datetime, nullable | Subscription expiry |
| `billing_provider` | str, nullable | mock, stripe, or paddle |

## Tier Gating

**File:** `backend/app/middleware/tier_gate.py`

The `TierGate` decorator enforces subscription requirements on endpoints:

```python
@router.post("/advanced-feature")
@requires_creator  # Returns 402 if tier < Creator
async def advanced_feature(...):
    ...
```

Returns `402 PAYMENT_REQUIRED` with an `X-Required-Tier` header when the tenant's subscription is insufficient.

### Surface Access

**File:** `backend/app/services/surface_access_service.py`

Maps tiers to available surfaces and usage limits:

| Tier | Surfaces | Projects | Scenes/Month | Video Renders/Month |
|------|----------|----------|-------------|-------------------|
| Explorer | Telegram | 1 | 10 | 0 |
| Creator | Telegram, Web | 5 | 100 | 10 |
| Showrunner | Telegram, Web, API, MCP | Unlimited | Unlimited | 50 |

## Tenant Resolution

**File:** `backend/app/core/tenant_resolver.py`

After authentication, the system resolves the user's email to a tenant:

```python
async def get_tenant_id_for_user(db: AsyncSession, user: dict) -> uuid.UUID:
    # SELECT id FROM tenants WHERE admin_email = user['email']
```

If no tenant exists, the `/auth/me` endpoint auto-provisions one (see [auth-system.md](auth-system.md)).

## Configuration

| Variable | Default | Purpose |
|----------|---------|---------|
| `CES_DATABASE_URL` | — | PostgreSQL connection string |
| `CES_NEO4J_URI` | — | Neo4j bolt URI |
| `CES_NEO4J_USER` | `"neo4j"` | Neo4j username |
| `CES_NEO4J_PASSWORD` | — | Neo4j password |

## Related Documents

- [Auth System](auth-system.md) — How users authenticate and tenants are provisioned
- [Data Model](data-model.md) — Tenant model, VaultEntry, enums
- [Architecture Overview](architecture-overview.md) — Where tenancy fits in the stack
