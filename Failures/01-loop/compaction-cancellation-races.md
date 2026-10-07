---
type: failure
concepts: [abort-propagation, auto-compaction, run-settlement]
harnesses: [pi]
---
**Symptom** — Cancellation could start an auto-compaction anyway, leave stale retry state, or be missed while waiting for summarization auth; RPC `abort` reported success while manual compaction kept running; manual and threshold compaction raced; tree navigation replaced an active compaction's UI; dispose left work running.

**Root cause** — Compaction/branch summary are a second "run" without the main loop's abort plumbing and mutual exclusion: abort classified by error-string matching (`message === "Compaction cancelled" || AbortError`), auth await not abortable, no abort checkpoints, "idle" excluded compaction.

**Fix · [[pi]]**
- `5b31ffd74` 2026-05-28 — dispose aborts agent/compaction/summary/retry/bash (`packages/coding-agent/src/core/agent-session.ts:1392-1413`) (#5029).
- `e56893f4c` 2026-08-03 — prevent auto-compaction race during manual compaction (#7370).
- `bea67d90d` 2026-09-03 — session abort cancels compaction; compaction + branch summary included in idle tracking (#8920).
- `e687434a6` 2026-09-07 — reject tree navigation during compaction (#9179).
- `de2de549b` 2026-09-19 — signal-based classification, abortable summarization auth, `throwIfAborted()` checkpoints (`agent-session.ts:3115-3119,3144,3174`) (#9340, #9777).

**Lesson** — Classify cancellation by signal state, never by error text; "idle" must include every background LLM operation; side phases need the same abort plumbing and mutual exclusion as the main loop.

Related: [[abort-propagation]] · [[auto-compaction]] · [[run-settlement]] · [[side-phase-input-lost]] · [[pi--abort-propagation|pi]]
