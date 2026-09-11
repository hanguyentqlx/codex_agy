# Test Plan

The architecture is accepted only when failure paths are tested, not just the happy path.

## Unit tests

- task state transition validation;
- failure classification;
- config parsing/validation;
- path canonicalization and traversal rejection;
- AGY output normalization;
- idempotency behavior;
- apply precondition checks.

## Integration tests

- MCP stdio discovery/initialize/tool calls;
- protocol capability negotiation and fallback;
- SQLite transaction/restart recovery;
- real Git repository/worktree creation and cleanup;
- worker process start/resume/cancel using a deterministic fake AGY CLI;
- verification runner timeout/output handling.

## End-to-end tests

Required scenarios:

1. `delegate_task` -> AGY edits -> verify -> review -> apply -> destination verify -> `APPLIED`.
2. Codex asks for changes -> `continue_task` resumes same conversation/worktree.
3. AGY exits non-zero -> evidence persists -> retry succeeds.
4. bridge crashes during worker run -> restart reconciles task/worktree.
5. destination branch advances before apply -> no silent overwrite.
6. cherry-pick conflict -> `APPLY_CONFLICT` and clean abort.
7. post-apply verification fails -> rollback policy executes and evidence remains.
8. duplicate mutating request -> idempotent result, no duplicate worker/apply.
9. forbidden command/path -> structured `POLICY` result.
10. cancellation -> process stops, task becomes `CANCELLED`, evidence preserved.

## Compatibility tests

Run conformance tests against the MCP protocol/capabilities supported by the installed Codex version. Do not assume a single protocol revision. Keep fixtures for current and fallback negotiation paths.

## Property/invariant tests

Assert globally:

- no two active mutation leases for one task;
- `APPLIED` implies destination verification passed;
- every apply has an immutable reviewed source commit;
- terminal cancellation cannot later mutate the worktree without explicit reopen/retry semantics;
- destination repo is never force-reset by the bridge.

## Performance targets for v1

Optimize correctness first. Practical goals:

- bridge idle footprint small enough for always-on local use;
- task metadata operations are sub-second;
- no polling loops faster than necessary;
- large worker stdout is streamed/capped rather than retained unbounded in memory.
