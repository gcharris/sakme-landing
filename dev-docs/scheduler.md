# Scheduler

> APScheduler integration, cron jobs, health checks, and execution logging.

## Overview

CES uses APScheduler 3.x with an `AsyncIOScheduler` for background task scheduling. Jobs persist to PostgreSQL via `SQLAlchemyJobStore`, surviving Cloud Run cold starts. Every job execution is logged to a `scheduled_job_logs` table with duration, status, and error tracking. Failed jobs trigger Telegram notifications to the Director.

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                 SchedulerService                      │
│                                                       │
│  APScheduler (AsyncIOScheduler)                       │
│     │                                                 │
│     ├── Cron Jobs     (CronTrigger)                  │
│     ├── Interval Jobs (IntervalTrigger)              │
│     └── Job Store     (SQLAlchemy → PostgreSQL)      │
│                                                       │
│  Every execution:                                     │
│     1. Log start to scheduled_job_logs               │
│     2. Run job function                               │
│     3. Log completion (duration, status, errors)     │
│     4. Notify Director on failure (Telegram)         │
└──────────────────────────────────────────────────────┘
```

## Key Service

**File:** `backend/app/services/scheduler_service.py`

```python
class SchedulerService:
    def __init__(self, session_factory: async_sessionmaker, settings):

    async def start(self) -> None       # Initialize and start scheduler
    async def stop(self) -> None        # Graceful shutdown
    def add_cron_job(self, job_id, func, name, cron_expression, ...)
    def add_interval_job(self, job_id, func, name, minutes=60, ...)
    async def list_jobs(self) -> list[dict]
    async def get_job_history(self, job_id=None, limit=20) -> list[dict]
    async def remove_job(self, job_id) -> bool
    async def pause_job(self, job_id) -> bool
    async def resume_job(self, job_id) -> bool
    async def trigger_job(self, job_id) -> bool
```

### Job Store Configuration

APScheduler 3.x requires a synchronous database URL for its job store. The service automatically converts `postgresql+asyncpg://` to `postgresql://`:

```python
if db_url.startswith("postgresql+asyncpg://"):
    job_store_url = "postgresql://" + db_url[len("postgresql+asyncpg://"):]
```

### Job Wrapping

All registered jobs are wrapped with execution logging:

1. Record start time in `scheduled_job_logs`
2. Execute the actual job function
3. Record end time, duration, and status
4. On failure: log error message and send Telegram notification

The wrapper handles both sync and async job functions.

### Default Configuration

```python
job_defaults = {
    "coalesce": True,        # Collapse missed executions into one
    "max_instances": 1,       # Prevent parallel runs of same job
    "misfire_grace_time": 300, # 5 minutes grace for Cloud Run cold starts
}
```

## Built-in Jobs

### System Heartbeat

Registered automatically on startup. Runs every 5 minutes. Validates the scheduler is alive and PostgreSQL is reachable.

```python
self.add_interval_job(
    job_id="system.heartbeat",
    func=self._heartbeat,
    name="System Heartbeat",
    minutes=5,
)
```

### Health Checks (TASK-D04)

**File:** `backend/app/services/health_check_service.py`

Periodic checks registered in the FastAPI lifespan:

| Job | Schedule | Checks |
|-----|----------|--------|
| `health.check` | Every 15 min | PostgreSQL, Neo4j, LLM provider connectivity |
| `health.weekly_summary` | Monday 8am UTC | 7-day aggregation from job logs |

### Other Registered Jobs

| Job | Service | Schedule |
|-----|---------|----------|
| Proactive notifications | `ProactiveNotificationService` | Cron (configurable) |
| Cost anomaly check | `CostDashboardService` | Every 60 min |
| AI Council debate | `AiCouncilService` | 2am UTC nightly |
| Morning brief | `SignalAggregationService` | 1pm UTC daily |
| Memory synthesis | `MemorySynthesisService` | Configurable |
| Standing order reminders | `StandingOrderService` | Cron per order |

## ScheduledJobLog Model

**File:** `backend/app/models/scheduled_job.py`

```python
class ScheduledJobLog(Base, UUIDMixin):
    job_id: str              # APScheduler job ID
    job_name: str            # Human-readable name
    trigger_type: str        # "cron", "interval", "manual"
    status: str              # "running", "success", "error"
    started_at: datetime
    finished_at: datetime | None
    duration_ms: int | None
    result_summary: str | None
    error_message: str | None
    notified: bool           # Was Director notified?
```

## REST API

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/v1/scheduler/jobs` | List all registered jobs |
| GET | `/api/v1/scheduler/history` | Get execution history |
| POST | `/api/v1/scheduler/jobs/{id}/pause` | Pause a job |
| POST | `/api/v1/scheduler/jobs/{id}/resume` | Resume a paused job |
| POST | `/api/v1/scheduler/jobs/{id}/trigger` | Trigger immediate execution |
| DELETE | `/api/v1/scheduler/jobs/{id}` | Remove a job |

## Feature Flag

The scheduler is disabled by default and must be explicitly enabled:

```
CES_SCHEDULER_ENABLED=true
```

When disabled, `SchedulerService` is not instantiated and no jobs run. Health checks and other periodic tasks simply do not execute.

## Configuration

| Variable | Default | Purpose |
|----------|---------|---------|
| `scheduler_enabled` | `false` | Enable/disable scheduler |
| `scheduler_job_store_url` | `""` | Override job store DB URL |
| `scheduler_misfire_grace_time` | `300` | Grace seconds for missed jobs |
| `health_check_interval_minutes` | `15` | Health check frequency |
| `health_check_timeout_seconds` | `30` | Per-service timeout |

## Related Documents

- [deployment.md](deployment.md) -- Cloud Run considerations for scheduling
- [configuration.md](configuration.md) -- All scheduler configuration
- [telegram-integration.md](telegram-integration.md) -- Failure notifications
