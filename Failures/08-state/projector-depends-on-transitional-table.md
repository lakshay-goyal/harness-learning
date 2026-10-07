---
type: failure
concepts: [event-sourced-session-store, session-migration]
harnesses: [opencode]
---
**Symptom** — Moving a session (to another directory/workspace) failed on databases that only had the v1 schema: the `Moved` event's projector also deleted the session's v2 Context Epoch row, and the `session_context_epoch` table did not exist there (symptom inferred from the fix's test; the commit has no description).

**Root cause** — Legacy and v2 runtimes share one SQLite database and one projector layer. A projector registered for a shared event touched a table that only the transitional v2 schema guarantees.

**Fix · [[opencode]]** — `d8bf79225f` 2026-08-13 (#42444) "preserve v1 database compatibility": removed `SessionContextEpoch.reset` from the `Moved` and workspace-move projectors (`packages/core/src/session/projector.ts:242-256` at HEAD); its test "projects moved sessions without the transitional context epoch table". Side effect: the spec still says a move clears the epoch (`CONTEXT.md:118`; `specs/v2/schema-changelog.md:794`) while `SessionContextEpoch.reset` now has no callers (`packages/core/src/session/context-epoch.ts:111`) → [[stale-runner-recreates-context-after-move]].

**Lesson** — Projectors shared by two runtimes on one database may only touch tables both schemas guarantee; runtime-specific cleanup belongs in that runtime's own handler.

Related: [[event-sourced-session-store]] · [[session-migration]] · [[strict-schema-rejects-legacy-records]] · [[opencode--event-sourced-session-store|opencode impl]]
