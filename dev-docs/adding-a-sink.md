# Adding a Sink

> How to add a new output destination using the BaseSink pattern.

## Overview

Sinks are output adapters that deliver creative content to external platforms. CES has 18 sink implementations covering text, video, audio, and ebook formats. All sinks inherit from `BaseSink` and are registered with the `SinkRouter` for automatic dispatch.

## Existing Sinks

**Directory:** `backend/app/services/sinks/`

| Sink | File | Format |
|------|------|--------|
| Higgsfield Video | `higgsfield_video_sink.py` | Video |
| Kling Video | `kling_video_sink.py` | Video |
| Aggregator Video | `aggregator_video_sink.py` | Video (Fal.ai/Runware) |
| Veo Video | `veo_video_sink.py` | Video (Google Veo) |
| Seedance Video | `seedance_video_sink.py` | Video (music-synced) |
| TikTok Video | `tiktok_video_sink.py` | Video (short-form) |
| YouTube Video | `youtube_video_sink.py` | Video (upload) |
| Moltbook | `moltbook_sink.py` | Social post |
| Medium | `medium_sink.py` | Blog article |
| Substack | `substack_sink.py` | Newsletter |
| Newsletter | `newsletter_sink.py` | Generic newsletter |
| KDP Ebook | `kdp_ebook_sink.py` | EPUB/KDP |
| ElevenLabs Audiobook | `elevenlabs_audiobook_sink.py` | M4B audiobook |
| Telegram | `../telegram_sink.py` | Notifications |

## Step 1: Implement BaseSink

**File:** `backend/app/services/sinks/base.py`

```python
from abc import ABC, abstractmethod
from enum import Enum
from typing import Any
from pydantic import BaseModel

class SinkResult(str, Enum):
    SUCCESS = "SUCCESS"
    RETRY = "RETRY"           # Transient failure, retry later
    FAILED = "FAILED"         # Permanent failure, don't retry
    RATE_LIMITED = "RATE_LIMITED"

class SinkResponse(BaseModel):
    result: SinkResult
    external_id: str | None = None       # ID from destination platform
    error_message: str | None = None
    retry_after_seconds: int | None = None
    metadata: dict[str, Any] = {}

class BaseSink(ABC):
    MAX_RETRIES = 3

    @abstractmethod
    async def publish(self, payload: dict[str, Any]) -> SinkResponse:
        """Publish content to the destination."""
        pass

    @abstractmethod
    def supports_hook(self, hook_id: str) -> bool:
        """Check if this sink supports the given OpenClaw hook."""
        pass
```

Create your sink in `backend/app/services/sinks/`:

```python
# backend/app/services/sinks/my_platform_sink.py

from app.services.sinks.base import BaseSink, SinkResponse, SinkResult

class MyPlatformSink(BaseSink):
    """Sink for publishing to MyPlatform."""

    def __init__(self, api_key: str, base_url: str = "https://api.myplatform.com"):
        self._api_key = api_key
        self._base_url = base_url

    async def publish(self, payload: dict) -> SinkResponse:
        content = payload.get("content", "")
        title = payload.get("title", "")

        try:
            # Call the platform API
            async with httpx.AsyncClient() as client:
                resp = await client.post(
                    f"{self._base_url}/v1/posts",
                    json={"title": title, "body": content},
                    headers={"Authorization": f"Bearer {self._api_key}"},
                    timeout=30.0,
                )
                resp.raise_for_status()
                data = resp.json()

            return SinkResponse(
                result=SinkResult.SUCCESS,
                external_id=data.get("id"),
                metadata={"url": data.get("url")},
            )
        except httpx.HTTPStatusError as e:
            if e.response.status_code == 429:
                retry_after = int(e.response.headers.get("Retry-After", 60))
                return SinkResponse(
                    result=SinkResult.RATE_LIMITED,
                    error_message="Rate limited",
                    retry_after_seconds=retry_after,
                )
            return SinkResponse(
                result=SinkResult.FAILED,
                error_message=f"HTTP {e.response.status_code}",
            )

    def supports_hook(self, hook_id: str) -> bool:
        return hook_id in {"publish_post", "share_content"}
```

## Step 2: Register with SinkRouter

**File:** `backend/app/services/sinks/router.py`

The `SinkRouter` maps `(TargetMedium, provider)` tuples to sink instances:

```python
class SinkRouter:
    def register_sink(self, medium: TargetMedium, sink: BaseSink, *, provider: str = "default"):
        self._sinks[(medium, provider)] = sink

    async def route(self, entry: OutboxEntry) -> SinkResponse:
        # Looks up by (target_medium, provider)
        # Falls back to (target_medium, "default")
```

Register your sink during application startup (typically in `main.py` or a factory function):

```python
from app.services.sinks.my_platform_sink import MyPlatformSink

sink_router.register_sink(
    medium=TargetMedium.MY_PLATFORM,
    sink=MyPlatformSink(api_key=settings.my_platform_api_key),
    provider="default",
)
```

## Step 3: Add TargetMedium Enum

**File:** `backend/app/schemas/outbox.py`

Add your platform to the `TargetMedium` enum:

```python
class TargetMedium(str, Enum):
    # ... existing entries ...
    MY_PLATFORM = "my_platform"
```

## Step 4: Add Configuration

**File:** `backend/app/core/config.py`

Add API credentials and settings:

```python
class Settings(BaseSettings):
    # ... existing settings ...
    my_platform_api_key: str = ""
    my_platform_base_url: str = "https://api.myplatform.com"
```

## Step 5: Add Tests

Create `backend/tests/services/sinks/test_my_platform_sink.py`:

```python
import pytest
from app.services.sinks.my_platform_sink import MyPlatformSink
from app.services.sinks.base import SinkResult

@pytest.mark.asyncio
async def test_publish_success():
    sink = MyPlatformSink(api_key="test-key")
    # Mock httpx client...
    response = await sink.publish({"content": "Hello", "title": "Test"})
    assert response.result == SinkResult.SUCCESS

@pytest.mark.asyncio
async def test_rate_limit_handling():
    # Test 429 response handling
    ...
```

## Key Design Patterns

1. **SinkResult enum**: Use `RETRY` for transient failures, `FAILED` for permanent ones, `RATE_LIMITED` for backoff.
2. **External ID**: Always return the platform's content ID in `external_id` for tracking.
3. **Metadata**: Put platform-specific response data (URLs, analytics) in `metadata`.
4. **Error messages**: Never expose raw API keys or internal details in error messages.
5. **Multi-provider**: Register multiple sinks for the same `TargetMedium` with different `provider` strings (e.g., multiple video providers).

## Dispatch Flow

Content reaches sinks via the Outbox pattern:

```
Service creates OutboxEntry (type=SOCIAL_DRAFT, target_medium=MY_PLATFORM)
    ▼
OutboxRelay picks up pending entries
    ▼
SinkRouter.route(entry) finds registered sink
    ▼
Sink.publish(payload) delivers content
    ▼
OutboxEntry status updated (delivered/failed)
```

## Related Documents

- [event-sourcing.md](event-sourcing.md) -- Outbox pattern and relay
- [configuration.md](configuration.md) -- Adding config variables
- [testing.md](testing.md) -- Test patterns
