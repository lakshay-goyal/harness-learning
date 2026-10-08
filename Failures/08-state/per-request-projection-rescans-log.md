---
type: failure
concepts: [context-projection, durable-execution]
harnesses: [pi]
---
**Symptom** — In pi-durable every model request re-read and re-derived the whole conversation transcript from storage (head marker, edits, tool-result reordering) — cost grew with history on every generation/tool round.

**Root cause** — Durable-first design: context is a pure derivation from committed entries with no in-memory cache, so the hot path paid full storage scans.

**Fix · [[pi]]**
- `68ccef176` 2026-10-06 reuse the scanned context range within a task invocation.
- `da866ada1` 2026-10-06 keep each conversation's last range in memory for `contextRetentionMs` (default 600 000, `packages/durable/src/harness/agent.ts:52-53,71`).
- `ae92585d3` 2026-10-06 settings getters that throw must not break the retention check.

**Lesson** — A durable-first design must add caches back on hot paths, keyed so committed history stays the source of truth.

Related: [[context-projection]] · [[durable-execution]] · [[quadratic-long-session-operations]] · [[pi--context-projection|pi]]
