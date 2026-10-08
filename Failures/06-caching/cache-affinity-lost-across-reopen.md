---
type: failure
concepts: [session-affinity-cache-routing, durable-execution]
harnesses: [pi]
---
**Symptom** — In pi-durable, provider cache/session affinity was lost whenever a conversation was reopened, reset or compacted: requests went out with a new (or no) session id, so Codex/OpenAI implicit caches keyed by `prompt_cache_key` started cold.

**Root cause** — The provider-facing session id was process state, not durable conversation state.

**Fix · [[pi]]** — `70eceaade` 2026-10-04 "persist provider session identities": conversation-scoped document `pi.provider {sessionId: uuidv7()}` with `fork:"initial"` ("every fork starts with a fresh identity instead of copying its parent"); legacy conversations get one migration commit; forwarded as `sessionId` by generation (`packages/durable/src/harness/provider.ts:6-39`; `harness/generation.ts:204`). Covered by real Codex cache reuse e2e `test/provider-session-cache-e2e.test.ts`. (Stable pi uses the persisted session file id, `sdk.ts:435`.)

**Lesson** — Cache-affinity keys are durable conversation state: persist them, keep them across restarts/compaction, and mint fresh ones for forks.

Related: [[session-affinity-cache-routing]] · [[durable-execution]] · [[session-fork]] · [[pi--session-affinity-cache-routing|pi]]
