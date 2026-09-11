# Security Model

## Trust boundaries

Codex may propose work, but the bridge enforces execution policy. AGY is treated as an untrusted executor with scoped access.

## Default-deny policy dimensions

- filesystem roots AGY may access;
- executable/command allowlist or risk classes;
- network access;
- environment variable exposure;
- secret file access;
- destructive Git operations;
- package manager install behavior;
- maximum runtime/output/resource limits.

## Filesystem

AGY receives the task worktree as its primary writable root. The bridge must canonicalize paths and reject path traversal/symlink escapes outside allowed roots.

## Secrets

Do not copy the full parent environment into AGY. Build an explicit environment. Redact likely secrets from structured logs and persisted transcripts. Secret access should be opt-in and task-scoped.

## Command policy

Classify commands:

- `SAFE_READ`: status, diff, grep, test discovery.
- `SAFE_BUILD`: test/typecheck/build within worktree.
- `MUTATING_LOCAL`: package install, code generation, formatting.
- `DESTRUCTIVE`: reset --hard, clean -fdx, filesystem delete outside generated temp paths.
- `PRIVILEGED`: sudo, system service changes, host configuration.

V1 should automatically permit the first two classes, make `MUTATING_LOCAL` configurable, and deny `DESTRUCTIVE`/`PRIVILEGED` by default.

## Git protections

AGY must not push, force-push, rewrite destination history, modify remotes, or apply directly to the destination branch. Safe Apply is owned by the bridge.

## Network

Network access is configurable. If enabled, persist which attempt used it. Do not assume network responses are deterministic for verification.

## Logging

Every task/attempt/apply log entry includes correlation IDs but excludes raw credentials. Keep enough evidence to debug without persisting unnecessary prompt secrets.

## Denied operation behavior

A denied action becomes a structured `POLICY` failure/result. Do not silently weaken policy to make the worker succeed.
