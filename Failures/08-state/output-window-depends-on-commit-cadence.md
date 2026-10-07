---
type: failure
concepts: [partial-message-persistence, durable-execution, tool-output-truncation]
harnesses: [pi]
---
**Symptom** — In pi-durable, tail-retained tool output (bash) came out differently depending on when progress commits happened: a progress snapshot compacted stored output to the kept window, which could shift where a later window's first line started.

**Root cause** — The windowing/truncation of streamed output was applied incrementally at commit time, so the result depended on commit cadence (a persistence artifact) instead of only on the output bytes.

**Fix · [[pi]]** — durable 1.0.3 (`packages/durable/CHANGELOG.md:74`): "Tail-retained tool output no longer depends on when progress commits happened". Related windowing: `cdf79797b` 2026-10-04 windowed shell output with exact `skipped{bytes,newlines}` counts; outputWindow `{maxBytes, maxLines, minIntervalMs 100, bytesPerSecond 100 KiB}` (`packages/durable/src/harness/tool.ts:198-206`, `output.ts:261`).

**Lesson** — Persisted partial output must be a deterministic function of the produced bytes, independent of commit/flush timing.

Related: [[partial-message-persistence]] · [[durable-execution]] · [[tool-output-truncation]] · [[pi--partial-message-persistence|pi]]
