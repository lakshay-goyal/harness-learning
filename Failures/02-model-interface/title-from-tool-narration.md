---
type: failure
concepts: [auxiliary-model-calls]
harnesses: [opencode]
---
**Symptom** — Sessions that started with a shell run, an @file mention or a subtask got titles about the tooling ("Called the Read tool…", tool names) or no title at all, because the title side call saw synthetic narration parts instead of the user's intent.

**Fix · [[opencode]]**
- `564143071e` 2025-09-06 title not generated when the first message is a shell invocation.
- `17e8322c29` 2025-11-28 rule "ignore tool execution messages" — reverted the same day `0e280017e6`.
- `5db78f20e9` 2026-01-06 subtask-only first messages: title from the subtask prompts instead (`packages/opencode/src/session/prompt.ts:212-224`).
- `fe57d7bb38` 2026-01-07 "Never include tool names" + "When a file is mentioned, focus on WHAT the user wants to do WITH the file" (`packages/opencode/src/agent/prompt/title.txt`).
- Only non-synthetic user messages count (`prompt.ts:201-205`).

**Lesson** — Side calls must be fed the user's intent, not the harness's synthetic narration; filter synthetic parts before the side model sees them.

Related: [[auxiliary-model-calls]] · [[synthetic-tool-call-injection]] · [[opencode--auxiliary-model-calls|opencode]]
