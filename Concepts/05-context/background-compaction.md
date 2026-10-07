---
type: concept
stage: compaction
tier: candidate
aliases: [backgroundTokens, thresholdCompaction, "blocking compaction", "conversation-owned compaction", stale]
harnesses: [pi]
---
Start summarizing at a soft threshold below the hard limit without blocking the agent, and splice the summary in at the next idle/boundary point; only block when the hard threshold is crossed.

## Why
- Blocking compaction stalls the user for a full summarization call exactly when context is large (slowest request).
- Starting early hides latency, but the agent keeps appending while the summary is generated, so the summary must be placed relative to a stable cut and dropped if a newer compaction already superseded it.

## Design space
- Blocking only ✔ pi coding-agent.
- Soft (background) + hard (blocking) thresholds ✔ pi durable: blocking above `window − reserve`, background above `window − reserve − backgroundTokens` (32768).
- Placement: append immediately when idle, else at next boundary, else settle `stale` ✔ pi durable (write submission); concurrent compactions ordered by cut position.
- Ownership: background task owned by conversation (survives the generation), blocking one owned by the generation (structured concurrency).
- Start only if a cut exists (`selectCut` ≠ undefined) and no compaction already listed ✔ pi durable.

## Implementations
- [[pi--background-compaction|pi]] — durable-only: `thresholdCompaction` returns blocking/background; background `pi.compaction` task places summary via write submission; coding-agent has no background mode.

## Failures
- (none mined; spec-listed footgun: compaction losing application edits placed mid-summary, `packages/durable/docs/spec.md:4620-4625`)

## Related
[[auto-compaction]] · [[durable-execution]] · [[task-owned-subagent]] · [[context-projection]] · [[compaction-cut-point]]
