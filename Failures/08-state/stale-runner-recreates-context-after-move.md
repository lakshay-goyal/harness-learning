---
type: failure
concepts: [event-sourced-session-store, transcript-carried-system-prompt, location-scoped-runtime]
harnesses: [opencode]
---
**Symptom** (design-time, recorded in the v2 changelog) — After a session moved to another directory, a runner still executing in the old location could initialize the Context Epoch with the old location's environment and instructions, giving the moved session stale privileged context.

**Root cause** — Context sources are location-scoped services; the epoch baseline is written once and reused verbatim. Without a fence, whichever runner reaches initialization first decides the baseline.

**Fix · [[opencode]]** — Spec: "Fence Context Epoch initialization against the authoritative Session Location so a concurrent old-Location runner cannot recreate stale privileged context after a move" (`specs/v2/schema-changelog.md:795`). Code: every provider turn re-reads the session and interrupts if `session.location` differs from the runner's location, before touching the epoch (`packages/core/src/session/runner/llm.ts:179-183`). The companion rule "Moving a Session clears its active Context Epoch" (`CONTEXT.md:118`) was undone by `d8bf79225f` 2026-08-13 ([[projector-depends-on-transitional-table]]), so at HEAD the destination keeps the old baseline and receives an "The environment you are running in is now:" update instead (inference).

**Lesson** — Re-check placement at every turn boundary before writing anything durable that depends on location, and keep spec and code in sync when a fix removes half of a two-part rule.

Related: [[transcript-carried-system-prompt]] · [[event-sourced-session-store]] · [[location-scoped-runtime]] · [[opencode--transcript-carried-system-prompt|opencode impl]]
