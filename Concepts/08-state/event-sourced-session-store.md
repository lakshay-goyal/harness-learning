---
type: concept
stage: state
tier: candidate
aliases: [EventV2, "session.next.*", event_sequence, DurableDefinitions, SessionProjector, "events.project", aggregate seq, commit hook, opencode.db]
harnesses: [opencode]
---
Each session state change is a typed, versioned event appended in one SQLite transaction that also runs the read-model projectors. Model context is derived from the projections.

## Why
- One write path for UI, API, sync and model context: everything reads projections, so they cannot drift from each other.
- Replay and remote sync need an ordering authority; wall-clock time is not one ("Wall-clock timestamps may collide or move backwards, so they are not safe pagination boundaries", opencode `specs/v2/schema-changelog.md:378`).
- Projector + event row in one transaction means a crash never leaves an event without its projection or vice versa.
- Streaming deltas are too many and too ephemeral to store; the store needs explicit full-value boundaries (opencode: "Stream fragments are live-only; Text.Ended is the replayable full-value boundary").
- Cost: projectors become schema-coupled code that every runtime sharing the DB must agree on ([[projector-depends-on-transitional-table]]).

## Design space
- **Log shape**: append-only JSONL tree (pi, [[session-tree]]) · mutable rows written directly · **event table + per-aggregate `seq` + projected tables** (opencode).
- **Projection timing**: in the same immediate transaction (opencode `event.ts`) · async consumers.
- **Event versioning**: `type.version` per definition, payload encoded before storage and decoded on replay (opencode).
- **Local side effects**: per-publish `commit(seq)` hook run atomically but never replayed (opencode, used to advance the context snapshot).
- **Durable vs live-only events**: deltas live-only, `*.Ended` durable (opencode v2) · persist every delta.
- **Ordering**: by aggregate `seq` (opencode v2) · by time then id (opencode legacy `isAfter`, [[reordered-context-misidentifies-latest-turn]]).
- **Execution identity**: none — recovery reasons from prompts, projected history and tool state (opencode v2 rejects a durable execution id) · checkpointed task identity ([[durable-execution]], pi-durable).
- **Pre-release schema churn**: reset experimental projections wholesale while preserving canonical rows (opencode `reset_v2_session_state`) vs migrate ([[session-migration]]).

## Implementations
- [[opencode--event-sourced-session-store|opencode]] — legacy: every `updateMessage`/`updatePart` publishes an event whose core projector upserts SQLite rows; v2: `EventV2.publish` allocates seq, runs projectors and commit hook, writes event + sequence rows in one immediate transaction; durable `session.next.*` inventory.

## Failures
- [[timestamp-ordered-history-pagination]]
- [[projector-depends-on-transitional-table]]
- [[stale-runner-recreates-context-after-move]]
- [[reordered-context-misidentifies-latest-turn]]
- [[update-before-snapshot-on-subscribe]]
- [[unbounded-subscriber-buffering]]

## Tradeoffs
- [[mid-run-user-input]]
- [[session-store-format]]

## Related
[[context-projection]] · [[partial-message-persistence]] · [[session-migration]] · [[durable-execution]] · [[replicated-state]] · [[session-tree]] · [[transcript-carried-system-prompt]] · [[workspace-snapshots]]
