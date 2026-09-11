# Architecture Decision Records (Condensed)

## ADR-001: Codex is supervisor, AGY is worker

**Decision:** Codex owns planning/review/apply decisions; AGY owns implementation attempts.

**Why:** minimizes expensive supervisor work while keeping one authority for correctness.

## ADR-002: MCP over stdio for local control plane

**Decision:** use MCP/JSON-RPC over stdio for the first implementation.

**Why:** both processes are local; stdio avoids a daemon port, auth layer, socket lifecycle, and network failure modes.

**Future:** add Streamable HTTP only if remote/multi-host execution becomes a requirement.

## ADR-003: Capability negotiation, not protocol hard-coding

**Decision:** discover/negotiate supported MCP capabilities and maintain a compatibility path for older supported Codex behavior.

**Why:** Codex and MCP evolve independently; business workflow should not depend on one protocol revision.

## ADR-004: Git is the code transport

**Decision:** AGY produces committed code in an isolated worktree. MCP results carry references/metadata, not entire source trees.

**Why:** Git gives diffability, reviewability, provenance, conflict detection, and rollback.

## ADR-005: SQLite is the durable workflow store

**Decision:** SQLite + WAL is authoritative for task/event metadata; Git remains authoritative for code.

**Why:** local single-machine deployment needs restart durability but not Redis/Postgres operational overhead.

## ADR-006: Worker details behind Adapter

**Decision:** all AGY command-line flags, parsing, process management, conversation resume, and provider-specific errors live in `AGYAdapter`.

**Why:** prevents vendor/CLI coupling from leaking into orchestration and enables future worker replacement.

## ADR-007: Commit/cherry-pick Safe Apply

**Decision:** apply the exact reviewed commit through Git, then verify destination.

**Rejected:** blind file copying.

**Why:** immutable reviewed artifact, clean history, conflict detection, deterministic rollback evidence.

## ADR-008: Default-deny execution policy

**Decision:** privileged/destructive operations are denied by default. Worker runs primarily inside its worktree with an explicit environment.

**Why:** an autonomous coding worker must not inherit unrestricted host authority.

## ADR-009: No distributed infrastructure in v1

**Decision:** no Redis, message broker, container orchestrator, or distributed scheduler.

**Why:** they add complexity without solving a current single-machine requirement.

## ADR-010: Event history plus current snapshot

**Decision:** persist both a query-friendly task snapshot and append-only task events.

**Why:** snapshot makes normal reads cheap; events improve recovery, debugging, and auditability.
