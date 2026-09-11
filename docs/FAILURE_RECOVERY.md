# Failure Recovery

The bridge is designed so process failure does not equal workflow loss.

## Persistence model

SQLite stores task snapshots plus an append-only event history. Recommended tables:

```text
tasks
task_events
conversations
workspaces
attempts
verification_runs
apply_attempts
locks
```

Use WAL mode and foreign keys. Every externally visible transition should be persisted transactionally.

## Restart recovery

On bridge startup:

1. Load non-terminal tasks.
2. Reconcile each recorded worktree with Git.
3. Detect stale process ownership/leases.
4. Mark interrupted attempts as recoverable infrastructure failures.
5. Preserve AGY `conversation_id` for resume.
6. Re-run only idempotent preparation steps automatically.
7. Never auto-apply after restart without confirming the recorded commit and destination base.

## Worker crash

If AGY exits unexpectedly:

- capture exit code/stdout/stderr metadata;
- classify as `PROVIDER` or `INFRASTRUCTURE` in the adapter;
- keep the worktree untouched;
- keep the resumable conversation reference if valid;
- permit explicit retry or `continue_task`.

## Bridge crash

The task state is recovered from SQLite + Git. No in-memory queue is authoritative.

## Timeout

Timeouts are attempt-scoped, not task-scoped. Cancelling a timed-out process must not automatically delete its worktree or conversation.

## Git divergence

Before apply, capture:

```text
source commit
destination HEAD
merge base
worktree dirty state
```

If destination changed, recompute applicability. Never silently overwrite destination files.

## Apply conflict

A cherry-pick conflict transitions to `APPLY_CONFLICT`. Abort the in-progress Git operation, retain evidence, and return control to Codex for review/rebase/fix.

## Post-apply verification failure

If destination verification fails:

1. record `APPLIED_VERIFICATION_FAILED`;
2. preserve logs and applied commit SHA;
3. prefer a Git revert of the applied commit if rollback is policy-enabled;
4. never perform ad-hoc file restoration from memory;
5. return the task to review only through an explicit recovery action.

## Idempotency

Mutating operations require an idempotency key or stable operation ID. Replaying a request after transport failure must return the previous result rather than duplicate work.

## Leases

Use short SQLite-backed leases for task mutation. Leases prevent two bridge instances/processes from mutating the same task, while expiry permits crash recovery.
