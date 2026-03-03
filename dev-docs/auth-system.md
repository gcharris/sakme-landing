# Auth System

> CES uses a four-tier authentication hierarchy: Director API Key, Dev Auth Token, Dev Mode Stub, and Firebase ID Token. Production auth flows through Firebase Admin SDK. The system auto-provisions tenants on first login.

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/core/security.py` | Token verification, 4-tier auth hierarchy |
| `backend/app/api/v1/auth.py` | `/auth/me`, `/auth/dev-login`, auto-provisioning |
| `backend/app/api/v1/auth_providers.py` | OAuth flows (Google, Pinterest), BYOK, Telegram linking |
| `backend/app/models/tenant.py` | Tenant model, VaultEntry model, enums |
| `backend/app/models/user.py` | User model (Firebase UID mapping) |
| `backend/app/core/tenant_resolver.py` | Email-to-tenant lookup |
| `backend/app/core/config.py` | Auth-related environment variables |
| `backend/app/main.py` | Firebase Admin SDK initialization, startup guardrails |
| `frontend/src/lib/firebase.ts` | Firebase client config |
| `frontend/src/stores/auth.context.ts` | Auth state, login/logout, token refresh |

## Authentication Hierarchy

`get_current_user()` in `security.py` checks credentials in this order. The first match wins:

### 1. Director API Key (Highest Priority)

For the Director (admin) to bypass Firebase entirely.

```
Header: X-Director-Key: {secret}
Config: CES_DIRECTOR_API_KEY
Returns: {uid: "director-admin", email: "carrharris@gmail.com", name: "Director"}
```

Used for Command Center access and Telegram admin operations.

### 2. Dev Auth Token

For beta testers. Token is a base64-encoded JSON payload with a known password gate.

```
Header: Authorization: Bearer devauth_{base64_json}
Endpoint: POST /api/v1/auth/dev-login {email, password}
Password: !Writer2020 (hardcoded)
Returns: {uid: "dev-{md5_hash[:12]}", email: email, name: derived}
```

The `dev-login` endpoint accepts any email with the correct password and returns a `devauth_` prefixed Bearer token.

### 3. Dev Mode Stub

Fallback when Firebase is not configured and dev auth is enabled.

```
Conditions: firebase_project_id is empty AND allow_dev_auth is True
Returns: {uid: "dev-user-001", email: "dev@local.test", name: "Dev User"}
```

Returns `503 SERVICE_UNAVAILABLE` if conditions are not met.

### 4. Firebase ID Token (Production)

Standard Firebase Authentication via Admin SDK.

```
Header: Authorization: Bearer {firebase_id_token}
Verification: firebase_admin.auth.verify_id_token(token)
Returns: {uid: firebase_uid, email: email, name: name}
```

Requires `CES_FIREBASE_PROJECT_ID` set (default: `"kognist"`) and GCP Application Default Credentials.

## Startup Guardrail

A safety valve prevents dev auth from running in production:

```python
# backend/app/main.py
def _validate_auth_configuration() -> None:
    if settings.environment != "development" and settings.allow_dev_auth:
        raise RuntimeError(
            "FATAL: CES_ALLOW_DEV_AUTH=true in non-development environment. "
            "This would bypass all authentication. Refusing to start."
        )
```

## Auth Flow

### Request Lifecycle

```
Frontend                    Backend
   │                           │
   │  GET /api/v1/auth/me      │
   │  Authorization: Bearer {t} │
   ├──────────────────────────►│
   │                           │  1. get_current_user() checks 4 tiers
   │                           │  2. Returns {uid, email, name}
   │                           │  3. Upsert User record (create if missing)
   │                           │  4. Lookup Tenant by admin_email
   │                           │  5. Auto-provision Tenant if none exists
   │                           │
   │  {user, tenant, permissions}│
   │◄──────────────────────────┤
```

### Auto-Provisioning (First Login)

When a user calls `GET /api/v1/auth/me` and no tenant exists for their email:

```python
tenant = Tenant(
    tenant_id=uuid.uuid4(),
    display_name=name or email.split("@")[0],
    admin_email=email,
    role=TenantRole.OPERATOR,
    tier=TenantTier.FREE,
    status=TenantStatus.ACTIVE,
)
```

Dev users (`uid` starts with `"dev-"`) follow the same path but fall back to a hardcoded Director tenant if the database is unavailable.

### Frontend Auth Store

**File:** `frontend/src/stores/auth.context.ts`

The `AuthProvider` component:

1. Listens to `onIdTokenChanged(firebaseAuth, ...)` for token refresh
2. Stores token in `localStorage` (`ces_auth_token`)
3. Calls `GET /api/v1/auth/me` to resolve user and tenant
4. Exposes `useAuth()` hook with: `user`, `login`, `signUp`, `loginWithGoogle`, `logout`, `refreshAuth`

**AuthUser interface:**

```typescript
interface AuthUser {
  uid: string;
  email: string | null;
  display_name: string | null;
  tenant: TenantInfo | null;
  is_director: boolean;
  is_system_owner: boolean;
  can_access_command_center: boolean;
  can_access_dashboard: boolean;
  can_access_workbench: boolean;
  can_edit_sot: boolean;              // PRO+ tier
  can_access_architect: boolean;       // SOVEREIGN tier only
}
```

### Firebase Configuration

**Frontend** (`frontend/src/lib/firebase.ts`):

| Vite Env Var | Purpose |
|---|---|
| `VITE_FIREBASE_API_KEY` | Firebase API key |
| `VITE_FIREBASE_AUTH_DOMAIN` | Firebase auth domain |
| `VITE_FIREBASE_PROJECT_ID` | Firebase project ID |
| `VITE_FIREBASE_STORAGE_BUCKET` | GCS bucket |
| `VITE_FIREBASE_MESSAGING_SENDER_ID` | FCM sender ID |
| `VITE_FIREBASE_APP_ID` | Firebase app ID |

**Backend** (`backend/app/main.py`):

Firebase Admin SDK initializes on startup using Application Default Credentials. If `CES_FIREBASE_PROJECT_ID` is empty, Firebase auth is disabled and the app logs a warning.

## OAuth Providers

**File:** `backend/app/api/v1/auth_providers.py`

### Google OAuth

```
GET  /api/v1/auth/google/initiate   → Redirect to Google consent
GET  /api/v1/auth/google/callback   → Exchange code for tokens
```

Scopes: `generative-language.tuning`, `cloud-platform` (Gemini BYOK access)

### Pinterest OAuth

```
GET  /api/v1/auth/pinterest/initiate  → Redirect to Pinterest consent
GET  /api/v1/auth/pinterest/callback  → Exchange code for tokens
```

Scopes: `boards:read`, `pins:read`, `user_accounts:read` (visual style analysis)

### BYOK (Bring Your Own Key)

```
POST   /api/v1/auth/byok              → Store encrypted API key
DELETE /api/v1/auth/byok/{provider_id} → Remove key
```

Keys are encrypted with Fernet (symmetric encryption) via `vault_crypto.py` and stored in the `VaultEntry` model.

### Telegram Linking

```
POST /api/v1/auth/telegram-link → Create 15-minute linking token
```

User sends `/start link_{token}` in Telegram to complete the link.

### OAuth State Routing

OAuth callbacks use an encoded state parameter to route back to the correct frontend:

```
State format: {source}_{identifier}_{nonce}
Examples:
  web_user-uid_abc123xyz    → Redirect to frontend onboarding page
  telegram_123456789_def456 → Send Telegram confirmation message
```

## API Key Storage (Vault)

**Model:** `VaultEntry` (45 supported key types)

| Category | Key Types |
|----------|-----------|
| **LLM Providers** | GOOGLE_GEMINI_API_KEY, ANTHROPIC_API_KEY, OPENAI_API_KEY, DEEPSEEK_API_KEY, XAI_API_KEY, MISTRAL_API_KEY |
| **Social** | TWITTER_BEARER, DISCORD_BOT_TOKEN, SLACK_BOT_TOKEN, TELEGRAM_BOT_TOKEN |
| **Services** | ELEVENLABS_API_KEY, PINTEREST_OAUTH, YOUTUBE_OAUTH, GMAIL_OAUTH |
| **Development** | GITHUB_PAT, GITHUB_REPO_URL |

## Endpoint Auth Pattern

Every authenticated endpoint uses the `get_current_user` dependency:

```python
@router.post("")
async def create_project(
    data: ProjectCreate,
    user: Annotated[dict, Depends(get_current_user)],  # ← Auth
    db: Annotated[AsyncSession, Depends(get_db)],
):
    project = await service.create_project(data, user["uid"])
```

The `user` dict always contains `{uid, email, name}` regardless of which auth tier matched.

## Configuration Reference

| Variable | Default | Purpose |
|----------|---------|---------|
| `CES_FIREBASE_PROJECT_ID` | `"kognist"` | GCP project for Firebase Admin SDK |
| `CES_ALLOW_DEV_AUTH` | `False` | Enable dev auth stub (dev only) |
| `CES_DIRECTOR_API_KEY` | `""` | Secret for Director bypass auth |
| `CES_ENVIRONMENT` | `"development"` | Environment (dev/staging/production) |
| `CES_GOOGLE_CLIENT_ID` | `""` | Google OAuth client ID |
| `CES_GOOGLE_CLIENT_SECRET` | `""` | Google OAuth secret |
| `CES_GOOGLE_CALLBACK_URL` | Cloud Run URL | Google OAuth callback |
| `CES_PINTEREST_CLIENT_ID` | `""` | Pinterest OAuth client ID |
| `CES_PINTEREST_CLIENT_SECRET` | `""` | Pinterest OAuth secret |
| `CES_PINTEREST_CALLBACK_URL` | Cloud Run URL | Pinterest callback |

## Related Documents

- [Multi-Tenancy](multi-tenancy.md) — How tenant_id flows through requests
- [Architecture Overview](architecture-overview.md) — System architecture
- [Data Model](data-model.md) — Tenant, User, VaultEntry models
