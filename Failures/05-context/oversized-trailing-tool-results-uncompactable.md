---
type: failure
concepts: [compaction-cut-point, auto-compaction]
harnesses: [pi]
---
**Symptom** — When the trailing tool results alone exceeded the keep budget (e.g. several large reads after one assistant call), mid-run compaction silently did nothing and the next provider request overflowed.

**Root cause** — Tool results are never legal cut points; the budget boundary fell inside them, `cutPoints.find(c => c >= i)` found nothing, and the fallback was `cutPoints[0]` — the *first* cut point, i.e. keep everything.

**Fix · [[pi]]** — `8bdcd4498` 2026-09-18 (#9740) "Fall back to the latest safe assistant boundary when trailing tool results exceed the retained-token budget, allowing older history to compact before the next provider request." — `?? cutPoints[cutPoints.length - 1]` (`packages/coding-agent/src/core/compaction/compaction.ts:826`; legacy `findCutPoint` `:476`); same fallback in durable `selectCut` (`packages/durable/src/harness/compaction.ts:267`).

**Lesson** — Cut-point search needs a fallback for "the tail alone is too big": keep less than the budget rather than everything.

Related: [[compaction-cut-point]] · [[auto-compaction]] · [[threshold-check-misses-post-tool-request]]
