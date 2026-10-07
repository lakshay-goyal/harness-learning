---
type: concept
stage: state
tier: candidate
aliases: [pi-durable, Pico5, Harness, tasks, pi.generation, pi.tool, pi.compaction, SQLite storage, JSONL storage, Durable Object storage, checkpointed-compaction-task, storage-atomic-commit-invariant, global-id-namespace]
harnesses: [pi]
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
- **Deliberately no execution identity**: durable events + projections only; crash recovery reasons from prompts, projected history, provider attempts and tool state, a `running` tool is never replayed, and post-crash continuation is deferred — "Do not introduce an enclosing durable execution identity solely to group these facts" (opencode v2, `specs/v2/todo.md:73-74`; not an implementation of this concept) → [[event-sourced-session-store]].

## Implementations
- [[pi--durable-execution|pi]] — `@earendil-works/pi-durable` (experimental, v1.0.4): tasks `pi.generation`/`pi.tool`/`pi.compaction`, Memory/SQLite/JSONL/DO storage, poisoning, blocked tasks; consumed only by experimental TUI/session workers.

## Failures
- [[unbounded-subscriber-buffering]]
- [[storage-queue-close-races]]
- [[per-request-projection-rescans-log]]
- [[output-window-depends-on-commit-cadence]]
- [[settled-tool-vanishes-before-placement]]

## Tradeoffs
- [[session-store-format]]

## Related
[[crash-safe-tool-replay]] · [[replicated-state]] · [[context-projection]] · [[partial-message-persistence]] · [[session-tree]] · [[task-owned-subagent]] · [[steering-queue]] · [[background-compaction]] · [[deferred-responses]] · [[client-server-session-split]] · [[remote-execution-env]] · [[spec-driven-agentic-development]]
