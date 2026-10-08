---
type: failure
concepts: [cache-retention-control, session-affinity-cache-routing]
harnesses: [pi]
---
**Symptom** — Compaction and branch-summary requests paid cache-write premiums for prompts that would never be reused, and — sharing the main session's routing/cache id — displaced the main conversation's cache affinity.

**Root cause** — Side requests inherited default retention (`short`, or `long` via `PI_CACHE_RETENTION`) and the session id used as `prompt_cache_key`/affinity header. OpenAI GPT-5.6+ writes implicit caches unless told `{mode:"explicit"}`.

**Fix · [[pi]]**
- `9b3a20591` 2026-07-22 "isolate summarization requests" — fresh routing session ids, caching disabled.
- `241431c69` 2026-07-23 (#6618) "don't cache write compaction or branch summaries" — `cacheRetention:"none"` + `sessionId: options.sessionId ?? uuidv7()` (HEAD `packages/coding-agent/src/core/compaction/compaction.ts:619-639`; AgentSession passes `sessionId: undefined`, `agent-session.ts:2739`); OpenAI Responses explicit-mode models get `prompt_cache_options:{mode:"explicit"}` (`packages/ai/src/api/openai-responses.ts:106-114`); Codex omits `prompt_cache_key` on none (`openai-codex-responses.ts:275`). Durable compaction same (`packages/durable/src/harness/compaction.ts:168`).
- Cache warmer only restarts from requests carrying the session's own id, so summaries never replace the warmed entry (`sdk.ts:420-431`).
- Related attempt: `cff1cf52c` 2026-08-18 "cache-friendly compaction primitives" (summarize via the cached provider-context prefix) reverted next day `8dab70281` (reason unverified).
- Open: Azure Responses still sends `prompt_cache_key` with `cacheRetention:"none"` (`azure-openai-responses.ts:202`, unverified whether intended).

**Lesson** — One-off side calls must not share cache identity with the main conversation and should opt out of cache writes.

Related: [[cache-retention-control]] · [[session-affinity-cache-routing]] · [[auto-compaction]] · [[branch-summary]] · [[pi--cache-retention-control|pi]]
