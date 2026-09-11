# Implementation Roadmap

## Phase 0 — Repository foundation

Goal: make the project understandable and testable before agent logic grows.

Deliverables:

- TypeScript project
- Bun or Node runtime decision
- lint/typecheck/test scripts
- `src/`, `tests/`, `docs/`
- structured logger
- task schema + validation

Exit criteria:

```text
typecheck PASS
tests PASS
build PASS
```

## Phase 1 — Minimal MCP bridge

Goal: Codex can call one AGY worker reliably.

Implement:

```text
worker_execute
worker_status
```

Features:

- MCP over stdio
- spawn AGY
- capture stdout/stderr/exit code
- timeout/cancellation primitives
- structured result normalization

Exit test:

```text
Codex -> MCP -> AGY -> answer -> MCP -> Codex
```

## Phase 2 — Durable task/session model

Goal: Codex and AGY can talk over multiple turns.

Implement:

```text
worker_start
worker_continue
worker_get_result
worker_cancel
```

Persist:

- `task_id`
- `session_id`
- state
- timestamps
- transcript/events

Exit test:

```text
Codex gives task
AGY implements
Codex reviews
Codex asks AGY to fix
AGY resumes same session
```

## Phase 3 — Git worktree isolation

Goal: AGY never needs to modify the primary workspace.

Implement:

- create task branch
- create isolated worktree
- record `base_commit`
- run AGY inside that worktree
- obtain diff/commit metadata
- cleanup policy

Exit test:

```text
main workspace remains unchanged while AGY works
```

## Phase 4 — Review and safe-apply gate

Goal: no worker code reaches the destination without review and verification.

Implement states:

```text
REVIEW_REQUIRED
READY_TO_APPLY
APPLYING
APPLY_CONFLICT
VERIFYING
APPLIED
APPLIED_VERIFICATION_FAILED
```

Checks:

- destination base commit
- dirty working tree
- conflict detection
- apply
- destination test/typecheck/build

## Phase 5 — Recovery and resilience

Goal: killing a terminal/process does not destroy task continuity.

Implement:

- crash-safe task persistence
- stale PID detection
- AGY conversation resume
- recover incomplete task
- deterministic worktree discovery
- protocol/provider/infrastructure failure classes

## Phase 6 — Supervisor policy

Goal: Codex uses AGY for cheaper implementation and only takes over when needed.

Suggested policy:

```text
1. Codex plans.
2. AGY implements.
3. Codex reviews.
4. AGY gets up to N repair attempts.
5. Codex takes over only for hard failures or architectural changes.
```

Possible escalation rules:

- same test fails 3 times
- AGY repeats the same patch
- protocol/provider failure persists
- large architecture decision required
- unsafe change touches protected paths

## Phase 7 — Observability / operator UX

Goal: make the two-agent workflow easy to watch.

Add:

- `codex-agy status`
- `codex-agy tasks`
- `codex-agy logs <task>`
- tmux 2–3 pane launcher
- live state display
- transcript viewer

Suggested terminal layout:

```text
┌─────────────────────┬─────────────────────┐
│ Codex Supervisor    │ AGY Worker          │
│                     │                     │
│ plans / reviews     │ edits / tests       │
├─────────────────────┴─────────────────────┤
│ codex-agy status / logs                   │
└───────────────────────────────────────────┘
```

## Phase 8 — Optional remote workers

Only after the local version is stable:

- MCP Streamable HTTP
- authentication
- remote worker registry
- multiple AGY workers
- queue/scheduler

Do not start here; local stdio keeps v1 much simpler.

## Recommended first milestone

Build only this vertical slice first:

```text
Codex
  -> worker_start
  -> worker_execute
  -> AGY changes isolated worktree
  -> tests
  -> structured result
  -> Codex review
  -> worker_continue
  -> READY_TO_APPLY
```

Once that flow is reliable, add Safe Apply and recovery.
