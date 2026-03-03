# Agentic Loop

> Runtime loop engine for tool-using agent behavior, retries, fallback, and history compaction.

## Where It Lives

- `backend/app/services/agentic_loop.py`
- `backend/app/services/history_manager.py`
- `backend/app/services/model_fallback_service.py`
- `backend/app/services/agent_stream.py`
- `backend/app/services/telegram_stream_consumer.py`

## What It Solves

The loop replaces fixed-round tool execution with:

- sequential execution of all tool calls returned by a model turn
- retry behavior for transient provider failures
- context overflow handling through history compaction
- optional model fallback chain
- optional streaming events for progressive UX

## Core Flow

`AgenticLoop.run(...)`:

1. append user message to history
2. call model with current history + tools
3. if no tool calls: return assistant text
4. if tool calls exist: execute each tool sequentially, append tool results
5. repeat until completion, max turns, steering interrupt, or max attempts

## Error Handling

Classified in-loop:

- context overflow: compact history and retry
- rate limit: exponential backoff and retry
- auth/model error: attempt fallback model (if configured)
- tool exceptions: converted to tool-result text (not raised upstream)

## Streaming Events

Emitted events include:

- `message_start`, `message_delta`, `message_end`
- `turn_start`, `turn_end`
- `tool_start`, `tool_end`
- `compaction_start`, `compaction_end`
- `fallback`, `error`

Telegram integration consumes these via `TelegramStreamConsumer` and edits a single message at controlled rate.

## Configuration Flags

From `backend/app/core/config.py`:

- `agentic_loop_enabled`
- `agentic_loop_max_attempts`
- `agentic_loop_max_turns`
- `agentic_loop_max_tool_result_chars`
- `agentic_loop_max_context_chars`
- `agent_streaming_enabled`
- `agent_fallback_enabled`
- `agent_fallback_chain_json`

## Operational Notes

- Keep tool outputs bounded; large results are truncated to protect context budget.
- Compaction keeps recent turns and summarizes older context.
- Fallback chain defaults are Gemini-first unless overridden.

## Quick Verification

```bash
cd /Users/gch2024/Dev/context-engine-studio/backend
python -m py_compile app/services/agentic_loop.py app/services/model_fallback_service.py app/services/agent_stream.py app/services/telegram_stream_consumer.py
pytest tests/services/test_agentic_loop.py tests/services/test_model_fallback_service.py tests/services/test_agent_stream.py tests/services/test_telegram_stream_consumer.py -v
```
