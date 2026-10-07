---
type: failure
concepts: [auto-retry-backoff]
harnesses: [pi]
---
**Symptom** — Three tool-use LLM calls in one run that each needed one 429 retry exhausted "3/3" retries and failed the run, although every retry succeeded.

**Root cause** — Retry counter scoped to the whole run, accumulated across LLM calls.

**Fix · [[pi]]** — `4f004adef` 2026-01-29 — reset counter on every successful (non-error) assistant `message_end`, emitting `auto_retry_end{success:true}` (`packages/coding-agent/src/core/agent-session.ts:1173-1182`) (#1019).

**Lesson** — Retry budgets are per request, not per user turn.

Related: [[auto-retry-backoff]] · [[pi--auto-retry-backoff|pi]]
