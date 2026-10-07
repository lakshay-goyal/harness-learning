---
type: failure
concepts: [auto-compaction, message-conversion-layer]
harnesses: [opencode]
---
**Symptom** — After automatic compaction the agent did nothing (or the provider rejected the request): the rebuilt transcript was only the summary, ending on an assistant message, so there was no user turn to answer.

**Root cause** — Compaction replaced history with `[summary]` (`msgs = [msg]`) and the loop continued as if the user's pending request were still there.

**Fix · [[opencode]]**
- `71076d5c68` 2025-09-17 (#2659) "add synthetic user prompt after session compaction": a resume user message is persisted after the summary.
- `020ee56f25` 2025-11-25 "dont auto continue if compaction was manual": `/compact` stops after summarizing.
- `0fd6f365be` 2026-02-10 continuation text gains an escape hatch: "Continue if you have next steps, or stop and ask for clarification if you are unsure how to proceed." (`packages/opencode/src/session/compaction.ts:527-531`, marked `metadata.compaction_continue`).
- `34e2429c49` 2026-04-13 plugin hook `experimental.compaction.autocontinue` can veto the synthetic turn (`compaction.ts:497-518`); `e83b22159d` 2026-04-15 the continuation is billed as agent-initiated for GitHub Copilot.
- v2 avoids the shape: the checkpoint is a user message and the runner replays the original pending turn (`packages/core/src/session/runner/llm.ts:224-225`).

**Lesson** — Compaction must leave a valid, resumable turn shape; if it ends on the summary, add an explicit (and attributable) continuation turn, and don't add it when the user only asked to compact.

Related: [[auto-compaction]] · [[message-conversion-layer]] · [[pre-prompt-compaction-replays-turn]] · [[opencode--auto-compaction|opencode impl]]
