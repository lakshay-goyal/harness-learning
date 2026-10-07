---
type: failure
concepts: [overflow-recovery, auto-compaction, context-overflow-detection]
harnesses: [pi]
---
**Symptom** — After switching from a smaller-context model (e.g. Opus) to a larger one (e.g. Codex), the old model's overflow error (or its huge usage) triggered compaction for the new model; also stale pre-compaction errors re-triggered compaction on the first prompt after compaction.

**Root cause** — Overflow/threshold checks used the last assistant message without asking which model produced it or whether it predates the latest compaction.

**Fix · [[pi]]** — `615ed0ae2` 2026-01-07 (#535) "Fix compaction UX oddities when model switching": `_modelForMessage` returns undefined for messages from another provider/model, `sameModel` gates overflow; skip when `assistant.timestamp <= latestCompaction.timestamp` (`packages/coding-agent/src/core/agent-session.ts:615-619`, `2957-2974` at HEAD; latest-vs-first compaction fix `7eb969ddb`). Comment: switching opus→codex "shouldn't trigger compaction for the new model".

**Lesson** — Judge overflow against the model (and context prefix) that produced the evidence, not the currently selected one.

Related: [[overflow-recovery]] · [[context-overflow-detection]] · [[stale-usage-drives-compaction]] · [[model-relabel-breaks-same-model-check]]
