# Protocol and Bridge API

## Principle

**MCP is the control plane. Git is the code plane.**

## Transport

V1 uses MCP JSON-RPC over `stdio` because Codex and the bridge run locally.

Do not hard-code one MCP revision. The server performs discovery/capability negotiation and chooses the strongest compatible behavior supported by the active Codex client. Keep a compatibility layer for older supported behavior.

## Separation of concerns

The protocol layer knows nothing about AGY command flags. It exchanges domain concepts such as tasks, feedback, results and cancellation.

AGY-specific details belong to `AGYAdapter`.

## Public business tools

### `delegate_task`

Creates and starts a durable task.

Example input:

```json
{
  "instruction": "Implement Google OAuth callback handling",
  "repository": "/repo",
  "verification_profile": "default",
  "idempotency_key": "req-123"
}
```

Example result:

```json
{
  "task_id": "task_01",
  "state": "RUNNING",
  "workspace_id": "wt_task_01"
}
```

### `continue_task`

Continues the same task after Codex feedback and reuses the worker conversation when available.

```json
{
  "task_id": "task_01",
  "feedback": "Fix the token expiry race and add a regression test.",
  "idempotency_key": "req-124"
}
```

### `inspect_task`

Returns the durable task snapshot plus latest useful result metadata.

```json
{
  "task_id": "task_01",
  "state": "REVIEW_REQUIRED",
  "conversation_id": "agy_conv_x",
  "workspace_id": "wt_task_01",
  "commit": "abc123",
  "summary": "Implemented callback handling",
  "verification": {"status": "passed"},
  "failure": null
}
```

### `cancel_task`

Cancels an active attempt and moves the task to `CANCELLED` according to the task state rules. Worktree/evidence retention follows policy.

## Standardized task lifecycle compatibility

If the negotiated MCP capabilities provide standardized task lifecycle primitives, map internal durable task operations onto them instead of duplicating semantics unnecessarily.

If the connected Codex version lacks those capabilities, expose equivalent behavior through the bridge tools above. The internal domain model remains the same in either case.

## Identifiers

Do not use a generic `session_id` for everything.

```text
task_id          durable business workflow
conversation_id  AGY conversation identity
workspace_id     worktree identity
attempt_id       process execution identity
apply_id         Safe Apply identity
```

MCP transport/process lifetime is intentionally not part of this identity model.

## Event envelope

Internal persisted events use a versioned envelope:

```json
{
  "schema_version": 1,
  "event_id": "evt_01",
  "task_id": "task_01",
  "attempt_id": "attempt_02",
  "timestamp": "2026-09-11T20:00:00+07:00",
  "type": "worker_result",
  "payload": {}
}
```

Representative types:

```text
task_created
workspace_prepared
worker_started
worker_finished
review_requested
review_feedback
state_changed
verification_started
verification_finished
apply_started
apply_finished
failure_recorded
task_cancelled
```

## Result layers

Keep three outputs distinct:

```text
human_summary       concise supervisor-readable summary
structured_result   source of truth for orchestration
raw_output          bounded diagnostic evidence
```

Critical state transitions never depend on parsing free-form prose in the orchestrator.

## Worker adapter normalization

The adapter returns typed data such as:

```json
{
  "conversation_id": "agy_conv_x",
  "exit_code": 0,
  "summary": "Implemented feature",
  "failure_class": null,
  "raw_output_ref": "log://attempt_02"
}
```

Provider/CLI peculiarities are normalized here, before reaching orchestration.

## Idempotency

Every mutating request accepts or derives a stable operation/idempotency key. Replayed calls after a transport interruption return the existing operation result rather than creating duplicate attempts/worktrees/applies.

## Cancellation

Cancellation is cooperative first and forceful second:

1. request graceful worker termination;
2. wait bounded grace period;
3. terminate process tree if necessary;
4. persist cancellation result;
5. never delete worktree/evidence as part of process kill itself.

## Error contract

Return structured failures containing:

```text
class
code
retryable
message
attempt_id
evidence refs
```

Classes are defined in [TASK_MODEL.md](TASK_MODEL.md).
