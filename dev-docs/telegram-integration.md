# Telegram Integration

> Bot setup, webhook handlers, callback system, Director tools, and agent routing.

## Overview

Telegram is the **primary Director interface** for CES. The Director manages the platform, reviews content, and interacts with the AI agent through a Telegram bot. There are two operational modes:

- **Metro Coding**: Approve/reject code changes from your phone via inline keyboards
- **Beach Coding**: Full conversational agent with 23+ tools via natural language

Telegram also serves as a notification channel for system health, creator activity, and proactive alerts.

## Architecture

```
Telegram ──► Webhook ──► telegram.py router
                              │
                   ┌──────────┴──────────┐
                   ▼                      ▼
            Callback Query          Text Message
            (button press)          (conversation)
                   │                      │
                   ▼                      ▼
            Handle action            AgentRouter
            (approve/reject/         (route to Director
             crystallize/etc.)        or User agent)
                   │                      │
                   ▼                      ▼
            OutboxEntry update      ArchitectAgent
            + TelegramSink reply    (25 tools, Gemini)
```

## Key Files

| File | Purpose |
|------|---------|
| `backend/app/api/v1/telegram.py` | Main webhook router (callbacks + messages) |
| `backend/app/api/v1/telegram_video.py` | Video-specific handlers (selection, render, refinement) |
| `backend/app/api/v1/telegram_admin.py` | Admin commands |
| `backend/app/api/v1/telegram_newsletter.py` | Newsletter handlers |
| `backend/app/services/telegram_sink.py` | Outbound message delivery |
| `backend/app/services/architect_agent.py` | Director agent (25 tools) |
| `backend/app/services/agent_router.py` | Routes to Director or User agent |

## TelegramSink

**File:** `backend/app/services/telegram_sink.py`

Handles all outbound messages to Telegram:

```python
class TelegramSink:
    async def send_merge_notification(self, ...)     # Code change approval
    async def send_review_notification(self, ...)    # Finding review
    async def send_health_notification(self, ...)    # System health alerts
```

### Forum Topic Routing

Messages are routed to specific forum topics based on type:

| Message Type | Config Variable | Purpose |
|-------------|----------------|---------|
| `creative` | `telegram_topic_creative` | Creative generation results |
| `system` | `telegram_topic_system` | System events, errors |
| `knowledge` | `telegram_topic_knowledge` | Codex and learning events |
| `settings` | `telegram_topic_settings` | Configuration changes |
| `health` | `telegram_topic_director_health` | Health check failures |
| `creator_activity` | `telegram_topic_director_creators` | Creator activity |
| `council` | `telegram_topic_director_council` | AI Council results |

Topic IDs of `0` disable routing and messages fall back to the main chat.

## Callback Handlers

Inline keyboard callbacks follow the pattern `action:entity_id`:

| Callback Pattern | Action |
|-----------------|--------|
| `approve:{id}` | Approve code change |
| `reject:{id}` | Reject code change |
| `crystallize:{id}` | Crystallize knowledge |
| `apply_fix:{id}` | Apply observation fix |
| `view_diff:{id}` | View observation diff |
| `dismiss:{id}` | Dismiss finding |
| `video_select:{id}` | Select video draft |
| `video_render:{id}` | Trigger final render |
| `video_refine:{id}` | Request video refinement |

Each callback updates the corresponding `OutboxEntry` status and sends a confirmation reply.

## ArchitectAgent (Director Tools)

**File:** `backend/app/services/architect_agent.py`

The Director's AI assistant with 25 Vertex AI function-calling tools:

### Platform Operations
| Tool | Purpose |
|------|---------|
| `search_moltbook` | Search posts by topic |
| `create_draft` | Create SOCIAL_DRAFT for HITL approval |
| `scan_mentions` | Scan for comments on posts |
| `dispatch_coding_task` | Send task to Claude Code |
| `get_pending_drafts` | List pending drafts |
| `get_my_posts` | Fetch recent posts |
| `get_active_conversations` | List trending conversations |

### Codebase Operations
| Tool | Purpose |
|------|---------|
| `read_project_file` | Read a file from the repo |
| `list_directory` | List files in a directory |
| `search_code` | Grep for patterns |
| `get_recent_commits` | Show recent git history |
| `check_logs` | Query Cloud Run logs |
| `run_observation` | Trigger security scan |

### GitHub Operations
| Tool | Purpose |
|------|---------|
| `create_branch` | Create a git branch |
| `edit_file_github` | Edit file on a branch |
| `create_pull_request` | Create PR for review |
| `web_search` | Search the web |
| `web_fetch` | Fetch and read a webpage |

### Model Preference
| Tool | Purpose |
|------|---------|
| `select_model` | Set preferred AI model |
| `get_current_model` | Show current model |
| `list_available_models` | List available models |

## AgentRouter

**File:** `backend/app/services/agent_router.py`

Routes incoming messages to the appropriate agent based on the sender:

- **Director** (matches `telegram_director_chat_id`): Routes to `ArchitectAgent`
- **Users** (TelegramUser linked to a tenant): Routes to `UserAgent`

Both agents receive `tenant_id` from the routing context so `PipelineContextBuilder` fires correctly.

## TelegramUser Model

Links Telegram accounts to CES tenants:

```python
class TelegramUser:
    telegram_id: str       # Telegram user ID
    tenant_id: UUID        # CES tenant
    user_id: UUID          # CES user
    username: str | None   # Telegram username
```

## Configuration

| Variable | Purpose |
|----------|---------|
| `CES_TELEGRAM_BOT_TOKEN` | Bot API token |
| `CES_TELEGRAM_DIRECTOR_CHAT_ID` | Director's Telegram chat ID |
| `CES_TELEGRAM_WEBHOOK_SECRET` | Webhook verification |
| `CES_TELEGRAM_TOPIC_*` | Forum topic IDs (7 topics) |

## Batch Processing

The `complete_task` and `update_task_status` tools accept comma-separated task IDs for batch operations, reducing the number of messages needed for bulk actions.

## Related Documents

- [auth-system.md](auth-system.md) -- Director API key and user authentication
- [configuration.md](configuration.md) -- Telegram configuration variables
- [event-sourcing.md](event-sourcing.md) -- OutboxEntry and HITL approval flow
