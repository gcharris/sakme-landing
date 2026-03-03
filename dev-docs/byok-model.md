# BYOK Model (Bring Your Own Key)

> BYOK architecture, provider routing, budget orchestration, and vault encryption.

## Overview

CES is designed as a **BYOK-first platform**: creators connect their own AI provider API keys, and all LLM calls route through their credentials. The platform provides the intelligence layer (soul, state, policy) that makes any model behave as a competent creative assistant. When no BYOK key is available, the platform's own credits are used, tracked via the budget orchestrator.

This architecture means CES customers bring their own compute. The platform's value is the Context Engine, not the model access.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Service Layer                      │
│   DirectorService, BrainstormService, etc.          │
│          │                                           │
│          ▼                                           │
│   BudgetOrchestrator.route_llm_action()             │
│          │                                           │
│          ├── BYOK? ──► VaultCrypto.decrypt(key)     │
│          │               └──► LLMService (user key)  │
│          │                                           │
│          └── Platform? ──► Budget check + reserve    │
│                            └──► LLMService (our key) │
│                                                      │
│   All calls logged as UsageRecord                    │
└─────────────────────────────────────────────────────┘
```

## BudgetOrchestrator

**File:** `backend/app/services/budget_orchestrator.py`

The **single gate** between all tenant-scoped services and the LLM layer. No service should import `LLMService` or `LLMClient` directly (enforced by CI lint check).

```python
class BudgetOrchestrator:
    def __init__(self, db_session_factory, model_pricing, settings, vault=None):

    async def route_llm_action(self, tenant_id, action_type) -> RoutingDecision
    async def record_actual_usage(self, tenant_id, decision, actual_input_tokens, actual_output_tokens, ...)
    async def get_budget_status(self, tenant_id) -> BudgetStatus
```

### RoutingDecision

Every LLM call goes through `route_llm_action()` which returns:

```python
@dataclass
class RoutingDecision:
    provider_id: str          # "gemini", "openai", "anthropic"
    model_id: str             # Specific model to use
    estimated_cost_usd: Decimal
    would_exceed_cap: bool    # Hard cap breach
    ux_hint: str              # User-facing message
    is_byok: bool             # Using user's own key
    credential_ref: str | None # Opaque vault reference (NEVER a raw key)
    routing_reason: str       # Why this routing was chosen
    reservation_id: UUID | None
    unmetered_fallback: bool
```

### Quality-Based Model Selection

The orchestrator maps `(action_type, subscription_tier)` to a quality level:

| Action | Explorer | Creator | Showrunner |
|--------|----------|---------|------------|
| Brainstorm | budget | budget | balanced |
| Generate Scene | budget | balanced | premium |
| Polish | budget | premium | premium |
| Voice Calibrate | budget | balanced | balanced |
| Score | budget | budget | budget |

Quality levels map to specific models via `ProviderConfig`.

### BYOK Auto-Detection

When the `VaultCrypto` instance is provided, the orchestrator checks for stored API keys:

1. Query `TenantRoutingPreference` for preferred provider
2. Query `VaultEntry` for the tenant's API key
3. If found, decrypt via `VaultCrypto` and return `is_byok=True`
4. BYOK calls skip budget checks (`BudgetOrchestrator` skips budget when `is_byok=True`)

### Google Free Tier

A special tier where users connect their Google OAuth and paste a free AI Studio API key. Only `gemini-2.0-flash` and `gemini-2.0-flash-lite` models are allowed:

```python
GOOGLE_FREE_ALLOWED_MODELS: set[str] = {
    "gemini-2.0-flash",
    "gemini-2.0-flash-lite",
}
```

If a Google Free user has no key configured, `GoogleFreeNoKeyError` is raised with setup instructions.

## VaultCrypto

**File:** `backend/app/core/vault_crypto.py`

Fernet-based symmetric encryption for API keys stored in `VaultEntry` rows:

```python
class VaultCrypto:
    def __init__(self, key: str | None = None):
        # Uses CES_VAULT_ENCRYPTION_KEY env var, or generates temp key in dev

    def encrypt(self, plaintext: str) -> str
    def decrypt(self, ciphertext: str) -> str
```

**Security notes:**
- Fernet provides authenticated encryption (AES-128-CBC + HMAC-SHA256)
- The encryption key **must** be set via `CES_VAULT_ENCRYPTION_KEY` in production
- In development, a temporary key is auto-generated (logged as WARNING)
- Raw API keys are **never** stored in `RoutingDecision` -- only opaque `credential_ref` values

## Budget Tracking

### Subscription Tiers

| Tier | Monthly Limit | Hard Cap |
|------|--------------|----------|
| `google_free` | $0 (user's quota) | N/A |
| `explorer` | $1.00 | Yes |
| `creator` | $5.00 | Yes |
| `showrunner` | $15.00 | No (soft) |

### UsageRecord Model

Every LLM call creates a `UsageRecord` with:
- `tenant_id`, `action_type`, `provider_id`, `model_id`
- `estimated_cost_usd`, `actual_input_tokens`, `actual_output_tokens`
- `status` (reserved, completed, failed)

### Credit Ledger

**File:** `backend/app/services/credit_ledger_service.py`

Double-entry credit tracking for purchased credits. The `CreditTransaction` model records debits and credits with full audit trail.

## Service Integration Pattern

Services use `_orchestrated_generate()` which tries the budget orchestrator first, falling back to direct `llm_client`:

```python
async def _orchestrated_generate(self, user_prompt, system_prompt, model, temperature, action_type, ...):
    if self._orchestrator and self._tenant_id and self._llm_service:
        decision = await self._orchestrator.route_llm_action(tenant_id, action_type)
        if decision.would_exceed_cap:
            raise BudgetExceededError(decision.ux_hint)
        response = await self._llm_service.generate_for_user(
            provider=decision.provider_id,
            model=decision.model_id,
            ...
        )
        await self._orchestrator.record_actual_usage(tenant_id, decision, ...)
        return response
    # Fallback: direct llm_client (no budget tracking)
    return await self.llm_client.generate(...)
```

## CI Enforcement

The CI pipeline (`ci.yml`) includes a check that flags unauthorized direct LLM imports. Services that legitimately bypass the orchestrator (infrastructure files, non-tenant-scoped services) are explicitly excluded.

## Related Documents

- [configuration.md](configuration.md) -- Budget and vault configuration variables
- [auth-system.md](auth-system.md) -- OAuth providers and BYOK key storage
- [deployment.md](deployment.md) -- Environment setup for vault encryption key
