---
type: failure
concepts: [auto-compaction, model-catalog]
harnesses: [opencode]
---
**Symptom** — The agent summarized on every turn: compaction fired, the next turn triggered it again, and the session never made progress.

**Root cause** — The early trigger was `(limit.context − limit.output) × 0.9`. Models whose catalog `limit.output` ≥ `limit.context`, or whose context was 0/unknown, made the threshold ≤ 0, so any token count "overflowed".

**Fix · [[opencode]]**
- `57b3051024` 2025-06-17 "fix agent getting caught in summary loop": guard `model.info.limit.context &&`.
- `f4c0d2d2fd` 2025-06-26 (#399) "guard against large output limit causing infinite summarize loop": `Math.max(…, 0)`.
- HEAD keeps both guards in a different form: `context === 0` → `isOverflow` false and `usable` 0; `usable = max(0, …)` (`packages/opencode/src/session/overflow.ts:10-34`). Side effect: custom models default to context 0 (`packages/opencode/src/provider/provider.ts:1610`), which silently disables threshold compaction instead of looping.

**Lesson** — Catalog limits are untrusted input: clamp every derived threshold and treat "unknown window" as an explicit mode, not as 0.

Related: [[auto-compaction]] · [[model-catalog]] · [[opencode--auto-compaction|opencode impl]] · [[overflow-recovery]]
