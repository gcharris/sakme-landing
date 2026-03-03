# Life-OS Chat Surfaces: Developer Guide

> How to connect Telegram, WhatsApp, or Discord to your self-hosted Life-OS instance.

## What This Is

Life-OS ships with a plugin-based chat surface architecture. You bring your own bot, point its webhook at your Life-OS backend, and every command routes through the same service layer that powers the Agent-C native app.

Telegram is the first implemented surface. WhatsApp and Discord follow the same pattern but aren't built yet — the architecture is ready, and this guide covers how to add them.

## Prerequisites

A running Life-OS backend. Either:

- **Tier 0 (Docker):** `git clone` + `docker-compose up -d` → backend at `http://localhost:8000`
- **Tier 1 Light (no Docker):** `pip install lifeos && lifeos config init` → SQLite + NetworkX, backend at `http://localhost:8000`

You also need a BYOK LLM key (Gemini free tier works) configured during setup.

## Telegram Setup

### 1. Create Your Bot

Open Telegram, message [@BotFather](https://t.me/BotFather):

```
/newbot
→ Choose a display name
→ Choose a username (must end in "bot")
→ BotFather gives you a token: 123456:ABC-DEF1234...
```

Save the token.

### 2. Get Your Chat ID

Send any message to your new bot in Telegram, then:

```bash
curl -s "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates" | python3 -m json.tool
```

Find `"chat": {"id": 987654321}` in the response. That's your Director chat ID.

### 3. Set Environment Variables

```bash
# Required
export CES_TELEGRAM_BOT_TOKEN="123456:ABC-DEF1234..."
export CES_TELEGRAM_DIRECTOR_CHAT_ID="987654321"
export CES_TELEGRAM_WEBHOOK_SECRET="$(openssl rand -hex 32)"

# Optional: organize messages into Telegram forum topics
export CES_TELEGRAM_TOPIC_CREATIVE=0
export CES_TELEGRAM_TOPIC_SYSTEM=0
export CES_TELEGRAM_TOPIC_KNOWLEDGE=0
```

### 4. Register the Webhook

Your backend needs a public HTTPS URL. For local development:

```bash
# Option A: ngrok
ngrok http 8000
# → Gives you https://abc123.ngrok.io

# Option B: Cloudflare Tunnel
cloudflared tunnel --url http://localhost:8000
# → Gives you https://your-tunnel.trycloudflare.com
```

Register with Telegram:

```bash
curl -X POST "https://api.telegram.org/bot<YOUR_TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://<YOUR_PUBLIC_URL>/api/v1/telegram/callback",
    "secret_token": "<YOUR_WEBHOOK_SECRET>"
  }'
```

You should get `{"ok": true, "result": true}`.

### 5. Start the Backend

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Send a message to your bot in Telegram. It should respond via the Agent-C agent, powered by your BYOK Gemini key.

### 6. Verify

```bash
# Check webhook status
curl -s "https://api.telegram.org/bot<YOUR_TOKEN>/getWebhookInfo" | python3 -m json.tool
```

Look for `"pending_update_count": 0` and no errors in `"last_error_message"`.

## How Telegram Routing Works

All Telegram updates arrive at a single endpoint:

```
POST /api/v1/telegram/callback
Header: X-Telegram-Bot-Api-Secret-Token: <webhook_secret>
```

The handler routes messages through a two-layer agent system:

- **Director messages** (matching `director_chat_id`) → `DirectorAgent` with 25+ tools (health, costs, tasks, observation, deployment)
- **Creator messages** (everyone else) → `UserAgent` with 12 tools (brainstorm, voice, project, generation)

Both agents use the same LLM backend. The difference is tool access, not intelligence.

### Command Priority

When a message arrives, the handler checks in order:

1. **Callback queries** — inline button presses (merge, reject, dismiss, format selection)
2. **Slash commands** — `/health`, `/brainstorm`, `/mirror`, etc.
3. **Free-text messages** — routed to the agent for conversational response

### Key Commands

**Director-only:**

| Command | What It Does |
|---------|-------------|
| `/health` | PostgreSQL + Neo4j + LLM provider checks |
| `/costs` | Daily spend trends, anomaly detection |
| `/observe` | Review code observation findings |
| `/tasks` | List and manage tasks |
| `/standing-orders` | View scheduled reminders |

**Creator-facing:**

| Command | What It Does |
|---------|-------------|
| `/start` | Initialize session, begin onboarding |
| `/brainstorm` | Start a brainstorm with Alter Ego |
| `/voice` | Run voice calibration tournament |
| `/mirror` | Creative development reflection |
| `/provenance` | Show growth edge patterns |
| `/project` | List or manage projects |
| `/crystallize` | Extract knowledge from a source |

## Agent Identity: The Soul Document

The agent's personality is defined by `config/soul-agent-c.md`. This is a markdown file that gets injected as the system prompt for every LLM call through the agent.

To customize, fork the file and point to your version:

```bash
export CES_ARCHITECT_SOUL_PATH="/path/to/your/soul.md"
```

The soul document defines: personality, safety rules (no unauthorized deletions, no publishing without approval), tool-use policy, and the boundary between Director and Creator interactions.

## Adding WhatsApp (Not Yet Built)

WhatsApp follows the same architecture. A developer would:

1. **Create** `backend/app/api/v1/whatsapp.py` — webhook handler
2. **Implement** the WhatsApp Business API webhook verification (GET for challenge, POST for messages)
3. **Route** incoming messages through the same `agent_router.py` that Telegram uses
4. **Set** environment variables:

```bash
export CES_WHATSAPP_PHONE_NUMBER_ID="..."
export CES_WHATSAPP_ACCESS_TOKEN="..."
export CES_WHATSAPP_VERIFY_TOKEN="..."
export CES_WHATSAPP_WEBHOOK_SECRET="..."
```

The service layer is surface-agnostic. `BrainstormService`, `VoiceCalibrationService`, `GovernorService` — they don't know or care whether the request came from Telegram, WhatsApp, REST, or the Capacitor app. The chat surface is just a thin translation layer between platform-specific message formats and the unified service API.

Estimated effort: ~200-300 LOC for a basic WhatsApp surface.

## Adding Discord (Not Yet Built)

Same pattern. A developer would:

1. **Create** a Discord bot at [discord.com/developers](https://discord.com/developers/applications)
2. **Create** `backend/app/api/v1/discord.py` — interaction handler
3. **Implement** Discord's interaction verification (Ed25519 signature check)
4. **Route** through `agent_router.py`
5. **Set** environment variables:

```bash
export CES_DISCORD_BOT_TOKEN="..."
export CES_DISCORD_APPLICATION_ID="..."
export CES_DISCORD_PUBLIC_KEY="..."
```

Discord has richer formatting options (embeds, components, threads) that could map well to tournament voting and project navigation. But the core routing is identical to Telegram.

Estimated effort: ~300-400 LOC for a basic Discord surface.

## Architecture Summary

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Telegram   │  │   WhatsApp   │  │   Discord    │
│   Webhook    │  │   Webhook    │  │  Interaction │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └────────────┬────┴────────┬────────┘
                    ▼             ▼
            ┌──────────────────────────┐
            │     agent_router.py      │
            │  Director? → DirectorAgent│
            │  Creator?  → UserAgent   │
            └────────────┬─────────────┘
                         ▼
            ┌──────────────────────────┐
            │    Service Layer         │
            │  (surface-agnostic)      │
            │  BrainstormService       │
            │  VoiceCalibrationService │
            │  GovernorService         │
            │  CreatorProfileService   │
            │  ...152+ services        │
            └──────────────────────────┘
```

Every chat surface is a thin adapter (~150-400 LOC) that translates platform-specific message formats into service calls. The intelligence lives in the service layer and the soul document, not in the chat handler.

## Troubleshooting

**Bot doesn't respond:**
- Check webhook is registered: `curl https://api.telegram.org/bot<TOKEN>/getWebhookInfo`
- Check backend logs for incoming requests at `/api/v1/telegram/callback`
- Verify `CES_TELEGRAM_BOT_TOKEN` matches BotFather's token exactly
- If using ngrok/cloudflare tunnel, verify the tunnel is still running

**Bot responds but agent is confused:**
- Check `CES_ARCHITECT_SOUL_PATH` points to a valid soul document
- Verify your BYOK Gemini key is set and has quota remaining
- Check `CES_TELEGRAM_DIRECTOR_CHAT_ID` — if it's wrong, you might be hitting the Creator agent when you expect the Director agent

**Webhook returns 403:**
- `CES_TELEGRAM_WEBHOOK_SECRET` must match the `secret_token` you registered with Telegram
- The header `X-Telegram-Bot-Api-Secret-Token` is checked on every request

**Commands not recognized:**
- Free-text routing catches everything that isn't a slash command — the agent handles it conversationally
- If a specific slash command doesn't work, check that the corresponding handler is registered in `telegram.py`
