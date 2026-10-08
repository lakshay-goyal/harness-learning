---
type: failure
concepts: [harness-evals]
harnesses: [pi]
---
**Symptom** — The eval validated the system prompt by reading it back from the session transcript; when core stopped persisting forced system prompts, the check would have validated the wrong prompt.

**Root cause** — Validation read a projection of state rather than what was actually sent to the provider. Core change: forced prompts were written as replacing system messages on every change and mid-conversation system messages were disabled, so `16292398a` stopped recording them.

**Fix · [[pi]]** — `5a3a03a7f` 2026-09-17 validated from transcript (`getCurrentSystemPrompt`); `16292398a` 2026-09-17 — the transform records what it SENT and validation uses that (`packages/evals/src/harness.ts:288-298,388-390`).

**Lesson** — Validate what was actually sent to the provider, not a stored projection of it.

Related: [[harness-evals]] · [[system-prompt-override]] · [[pi--harness-evals|pi]]
