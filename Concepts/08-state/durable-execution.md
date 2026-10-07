---
type: concept
stage: state
tier: candidate
aliases: [pi-durable, Pico5, Harness, tasks, pi.generation, pi.tool, pi.compaction, SQLite storage, JSONL storage, Durable Object storage, checkpointed-compaction-task, storage-atomic-commit-invariant, global-id-namespace, "Op::SuspendTurnAndShutdown", "Op::RecoverTurn", recover_turn_if_idle, "TurnStartKind::Recovery", RecordedTurnInput, daemon_recovery]
harnesses: [pi, codex]
---
Every agent-loop step (model request, tool call, compaction) is a persisted task state machine whose phase is committed to storage before its effects are shown; a crashed or restarted process reopens storage and resumes the interrupted work.

## Why
- In-memory loops lose the in-flight turn on crash, laptop sleep, or deploy; long-running/background work and remote presentation need the agent to outlive any one process or client.
- "Only committed state is observable": UIs never show progress that a crash could take back.
- Exposes the hard part: external effects are not idempotent — a tool interrupted mid-run may have partially happened ([[crash-safe-tool-replay]]).

## Design space
- **In-memory loop + append-only log of finished messages** (pi stable `packages/agent` + JSONL) vs **checkpointed task runtime** (pi-durable).
- **Storage**: memory (reference semantics), SQLite WAL, JSONL main + sidecars with commit markers, Cloudflare Durable Object SQLite.
- **Atomicity**: one commit line across entries, tasks, submissions, documents; uncertain storage failure poisons the session.
- **Effects**: effect sandwich (intent → effect → outcome) with per-tool replay policy `safe|unsafe` (default unsafe ⇒ "interrupted" result).
- **Partial output**: committed every 100 ms throttle window.
- **Missing code on reopen**: task stays pending `blocked` (missing_task / task_too_old / migration_failed), never terminalized.
- **Ownership**: structured concurrency tree, abort flows down ([[task-owned-subagent]]).
- **Writers**: single process owns storage (no cross-process locking; external `proper-lockfile`).
- **Turn-granular suspend/recover** ✔ codex: flush → cancel without terminal event → another worker recovers same turn id; requires input-persisted marker; re-samples instead of replaying tools.

## Implementations
- [[pi--durable-execution|pi]] — `@earendil-works/pi-durable` (experimental, v1.0.4): tasks `pi.generation`/`pi.tool`/`pi.compaction`, Memory/SQLite/JSONL/DO storage, poisoning, blocked tasks; consumed only by experimental TUI/session workers.
- [[codex--durable-execution|codex]] — turn-granular: suspend an unfinished root turn (flush first, no terminal event), recover it under the same turn id after a managed daemon restart by re-sampling from persisted history; no per-tool replay policy.

## Failures
- [[storage-queue-close-races]]
- [[per-request-projection-rescans-log]]
- [[output-window-depends-on-commit-cadence]]
- [[settled-tool-vanishes-before-placement]]
- [[cache-affinity-lost-across-reopen]] (06-caching) — In pi-durable, provider cache/session affinity was lost whenever a conversation was reopened, reset or…
- [[unbounded-subscriber-buffering]] (08-state) — Chord provider subscriptions invoked listeners inline while publishing (reentrant publication could reorder…

## Related
[[crash-safe-tool-replay]] · [[replicated-state]] · [[context-projection]] · [[partial-message-persistence]] · [[session-tree]] · [[task-owned-subagent]] · [[steering-queue]] · [[background-compaction]] · [[deferred-responses]] · [[client-server-session-split]] · [[remote-execution-env]] · [[spec-driven-agentic-development]] · [[sqlite-session-index]] · [[session-log-shape]]
