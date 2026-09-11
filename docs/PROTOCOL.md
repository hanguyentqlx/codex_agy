# Communication Protocol

## Principle

**MCP controls the worker; Git carries source changes.**

## Transport

For the first version, use MCP JSON-RPC over `stdio`.

Reasons:

- Codex and the bridge are on the same machine.
- No listening TCP port is needed.
- Lifecycle is tied to the parent process.
- Lower operational complexity than HTTP/WebSocket.
- Easy to migrate later because transport is hidden behind MCP.

## Message envelope

All task events should have a stable envelope:

```json
{
  "schema_version": 1,
  "event_id": "evt-...",
  "task_id": "task-...",
  "session_id": "agy-...",
  "timestamp": "2026-09-11T20:00:00+07:00",
  "type": "worker_result",
  "payload": {}
}
```

Suggested event types:

```text
task_created
worker_started
worker_output
worker_result
review_feedback
state_changed
verification_started
verification_result
worker_failed
worker_cancelled
```

## Session behavior

A `task_id` represents the engineering task.

A `session_id` represents the AGY conversational context.

Normally:

```text
1 task_id -> 1 long-lived AGY session_id
```

Codex review feedback must call `worker_continue(task_id, ...)` so the same AGY context is reused.

## Output separation

Keep these distinct:

```text
human_summary      short text for Codex/user
structured_result  machine-readable task state
raw_transcript     optional diagnostic log
```

Do not use raw CLI output as the source of truth for task state.

## Heartbeat / process status

For long-running tasks, `worker_status` should expose:

```json
{
  "state": "RUNNING",
  "pid": 12345,
  "session_id": "agy-abc",
  "last_activity_at": "...",
  "worktree": "...",
  "last_event": "running tests"
}
```

## Resume after terminal closes

Persist task metadata before AGY starts.

On restart:

1. Load incomplete tasks.
2. Check whether worker PID/session still exists.
3. Check worktree state.
4. Reattach/resume AGY conversation when supported.
5. Otherwise start a recovery turn that includes prior task metadata and latest result.

This makes terminal closure a recoverable infrastructure event rather than loss of the whole task.
