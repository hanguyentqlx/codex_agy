# Configuration

Configuration is layered and explicit:

1. built-in safe defaults;
2. project config;
3. environment overrides for deployment-specific values;
4. per-task options for a small allowlisted subset.

Recommended project file: `.codex-agy/config.toml`.

```toml
[bridge]
transport = "stdio"
database = ".codex-agy/state.db"
log_level = "info"

[agy]
command = "agy"
default_timeout_seconds = 1800
max_parallel_workers = 1

[workspace]
root = ".codex-agy/worktrees"
retain_failed = true
retain_applied = false

[verification]
commands = ["bun test", "bun run typecheck", "bun run build"]
stop_on_failure = true

[policy]
network = false
allow_package_install = false
allow_privileged_commands = false

[apply]
strategy = "cherry-pick"
auto_rollback_on_verification_failure = true
```

## Rules

- Resolve all relative paths from canonical repository root.
- Validate config before MCP server starts accepting mutations.
- Never put secrets in the project config committed to Git.
- Unknown keys should warn or fail in strict mode rather than being silently ignored.
- Persist the effective non-secret config fingerprint on each attempt for reproducibility.

## Worker portability

Only `AGYAdapter` may depend on AGY-specific flags/output/session semantics. The orchestrator consumes normalized adapter methods and errors.

Proposed adapter contract:

```ts
interface WorkerAdapter {
  start(input: WorkerStartInput): Promise<WorkerResult>;
  resume(input: WorkerResumeInput): Promise<WorkerResult>;
  cancel(attemptId: string): Promise<void>;
  probe(): Promise<WorkerCapabilities>;
}
```

This allows future adapters for other coding CLIs without changing Codex-facing MCP tools.
