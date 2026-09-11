# Implementation Roadmap v2

The project should be built as thin vertical slices. Do not implement every subsystem before proving the end-to-end path.

## Phase 0 — Foundation and contracts

Deliver:

- TypeScript + Bun project;
- source layout from `README.md`;
- schema validation for task/result/error/config;
- structured logger with correlation IDs;
- SQLite migration framework;
- deterministic fake AGY executable for tests;
- MCP capability/discovery spike against installed Codex.

Exit criteria:

```text
bun test PASS
bun run typecheck PASS
bun run build PASS
fake worker probe PASS
```

## Phase 1 — Minimal end-to-end delegation

Implement:

```text
delegate_task
inspect_task
```

Flow:

```text
Codex -> MCP stdio -> bridge -> fake/real AGY -> normalized result -> Codex
```

Include protocol negotiation, timeout and bounded output from day one.

## Phase 2 — Durable state + SQLite

Implement tables/events for:

```text
tasks
task_events
attempts
conversations
workspaces
```

Add idempotency keys and mutation leases.

Exit test: kill/restart bridge and recover a non-terminal task snapshot correctly.

## Phase 3 — Worktree isolation

Implement canonical repo identity, task branch/worktree lifecycle, base SHA tracking, cleanup policy and dirty-destination checks.

Exit test: AGY can modify/test/commit while the destination workspace remains unchanged.

## Phase 4 — Multi-turn worker conversations

Implement:

```text
continue_task
cancel_task
```

`AGYAdapter` owns all conversation resume semantics.

Exit test:

```text
Codex delegates
AGY commits
Codex reviews
Codex requests fix
AGY resumes same conversation/worktree
AGY produces updated reviewed commit
```

## Phase 5 — Verification gate

Add configurable verification profiles and persisted command results.

Requirements:

- worktree verification before review-ready result;
- bounded runtime/output;
- clear failure classification;
- no `READY_TO_APPLY` when required checks fail.

## Phase 6 — Transactional Safe Apply

Implement exact reviewed-commit validation and cherry-pick based apply.

Add tables/events for `apply_attempts` and destination verification.

Exit tests:

- successful apply;
- destination advanced;
- dirty destination rejection;
- cherry-pick conflict;
- destination verification failure + configured rollback.

## Phase 7 — Recovery and reconciliation

Implement startup reconciliation across SQLite, Git and process state.

Test crashes at boundaries:

- after event persisted, before process start;
- process started, bridge crashes;
- worker commit created, result not persisted;
- cherry-pick applied, bridge crashes before verification record;
- verification complete, response transport lost.

Recovery must be deterministic and idempotent.

## Phase 8 — Security hardening

Implement path canonicalization, explicit worker environment, command risk policy, secret redaction, protected Git operations, resource/output limits and optional network denial.

Add negative tests for traversal, symlink escape and prohibited commands.

## Phase 9 — Supervisor cost/escalation policy

Only after the mechanics are reliable, optimize expensive supervisor usage.

Suggested behavior:

```text
Codex plans once
AGY implements
Codex reviews compact diff/result
AGY gets bounded repair attempts
Codex takes over only for architecture/hard repeated failures
```

Escalation signals may include repeated identical failure, no-progress patches, protected-path changes, policy failures, or architecture-level uncertainty.

## Phase 10 — Operator UX

Add CLI/TUI helpers:

```text
codex-agy doctor
codex-agy status
codex-agy tasks
codex-agy inspect <task>
codex-agy logs <task>
codex-agy recover
codex-agy cleanup
```

Optional tmux layout may show Codex, AGY, and status/log panes, but UI must not become a dependency of the orchestration core.

## Phase 11 — Performance and controlled parallelism

Profile before optimizing.

Potential improvements:

- prepared worktree cache only if measured useful;
- SQLite indexes from real query patterns;
- configurable parallel workers for independent tasks;
- output streaming/backpressure;
- task log compaction/retention.

Keep Safe Apply serialized per destination repo.

## Phase 12 — Optional remote execution

Only if a real requirement appears:

- Streamable HTTP transport;
- authentication/authorization;
- remote worker registry;
- remote workspace/artifact model;
- scheduler/queue if multiple hosts require it.

Do not introduce Redis/brokers merely in anticipation of this phase.

## First production-worthy milestone

The first milestone is not “AGY can answer Codex”. It is this complete slice:

```text
Codex delegates
 -> durable task
 -> isolated worktree
 -> AGY implementation
 -> verification
 -> committed result
 -> Codex review
 -> one correction turn
 -> exact commit approval
 -> safe cherry-pick
 -> destination verification
 -> APPLIED
```

It must also survive a bridge restart before being considered reliable.
