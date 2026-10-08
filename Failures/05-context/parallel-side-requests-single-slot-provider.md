---
type: failure
concepts: [split-turn-summary, auto-compaction]
harnesses: [pi]
---
**Symptom** — Split-turn compaction issued two overlapping summary generations (history + turn prefix); single-concurrency local providers returned 429 and the compaction failed.

**Root cause** — The two summaries were generated with `Promise.all` for latency, ignoring the provider's concurrency limit (often 1 for local models).

**Fix · [[pi]]** — `f58c11562` 2026-07-01 (closes #5536) "serialize split-turn compaction summaries": history summary first, then turn prefix (`packages/coding-agent/src/core/compaction/compaction.ts:994-1033`; also `packages/agent/src/harness/compaction/compaction.ts:655-670` at `f58c11562` — historical: file added `83599e789`, removed with the experimental harness in `7fd478a2e`).

**Lesson** — Fan-out of side requests must respect the provider's concurrency, which for local models is often 1; default to sequential.

Related: [[split-turn-summary]] · [[auto-compaction]] · [[subagent-config-not-inherited]]
