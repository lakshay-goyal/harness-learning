---
type: failure
concepts: [runtime-plugin-loading, sdk-embedding]
harnesses: [pi]
---
**Symptom** — After `ctx.newSession()`, `fork()`, `switchSession()` or `reload()`, extension code kept using the `pi`/`ctx` it had captured; calls silently targeted the old (disposed) session — messages sent to the wrong session, state read from the wrong branch (#2860, #3606).

**Root cause** — Session replacement swaps the `AgentSession` object, but handles given to plugins were plain references with no generation check.

**Fix · [[pi]]**
- `1cc303d05` 2026-04-22 — replacement-session callbacks: `newSession/fork/switchSession({withSession})` hand a fresh `ReplacedSessionContext` (`packages/coding-agent/src/core/extensions/types.ts:398-439`).
- `f0cf8a59d` 2026-04-23 — old contexts invalidated; any use throws "This extension ctx is stale after session replacement or reload… move post-replacement work into withSession…" (`packages/coding-agent/src/core/extensions/loader.ts:192-200`).
- SDK analogue: subscriptions belong to the old `AgentSession` and must be rebound after `AgentSessionRuntime` replacement (`packages/coding-agent/docs/sdk.md:60`).

**Lesson** — Handles that outlive their generation must fail loudly; give plugins an explicit callback for "continue in the new session" rather than mutating references in place.

Related: [[runtime-plugin-loading]] · [[sdk-embedding]] · [[session-fork]] · [[pi--runtime-plugin-loading|pi]]
