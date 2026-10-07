---
type: failure
concepts: [overflow-recovery, auto-compaction]
harnesses: [pi, opencode]
---
**Symptom** — One context overflow triggered compact → retry → overflow → compact … indefinitely.

**Root cause** — Compact-and-retry had no bound; when compaction can't shrink the request enough (huge kept tail, oversized image, tiny window) every retry overflows again.

**Fix · [[pi]]** — `6b4b92042` 2026-03-03 (#1319) "stop overflow auto-compaction cascades": one-shot `_overflowRecoveryAttempted` latch, set before the recovery compaction, reset on any user `message_start` or any assistant `message_end` whose stopReason ∉ {error, length} (`packages/coding-agent/src/core/agent-session.ts:405`, `1119`, `1169-1171`, `3018-3045` at HEAD). Second overflow emits "Context overflow recovery failed after one compact-and-retry attempt. Try reducing context or switching to a larger-context model." + `session_compact_failed`. Durable equivalent: one overflow compaction per generation (`compacted` checkpoint, `packages/durable/src/harness/generation.ts:464-471`).

**Fix · [[opencode]]** `be20f865ac` 2026-03-02 (#14707): legacy overflow compaction holds out the triggering user turn and replays it after the summary; if the compaction request itself overflows → `ContextOverflowError` "Session too large to compact…" and `return "stop"` (`packages/opencode/src/session/compaction.ts:450-459`). v2 `820c984d47` 2026-06-05 (#31005): the post-compaction retry runs through `runAfterOverflowCompaction`, which has no recovery and dies with "Post-compaction provider attempt cannot recover another overflow" (`packages/core/src/session/runner/llm.ts:364-376`).

**Lesson** — Every self-healing loop needs a latch: one recovery attempt per user input, then surface the error.

Related: [[overflow-recovery]] · [[auto-compaction]] · [[context-overflow-detection]] · [[completed-response-retried-after-overflow]]
