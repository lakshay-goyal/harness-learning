---
type: implementation
harness: opencode
concept: iterative-summary-update
commit: ecc4916b5a
files: [packages/opencode/src/session/compaction.ts:97-113, packages/opencode/src/session/compaction.ts:363-371, packages/core/src/session/compaction.ts:47-55, packages/core/src/session/compaction.ts:160-174, packages/core/src/session/compaction.ts:183-188]
---
[[iterative-summary-update]] in [[opencode]].

## Mechanism
- Shared prompt builder `buildPrompt({previousSummary, context})` in `packages/core/src/session/compaction.ts:160-174`, imported by the legacy runtime. Order: `<conversation>` first, then `<prior-summary>`, then `SUMMARY_UPDATE_INSTRUCTIONS`, then the template ([[opencode--structured-compaction-summary]]).
- Update rules (`:47-55`): "The <prior-summary> is discarded after this: anything you do not carry into the new summary is lost"; carry forward objectives, constraints, user directives, decisions, parallel workstreams "even when the <conversation> does not mention them"; "Where they conflict, the conversation wins"; move Active → Completed; update Objective and Next Move.

### Legacy runtime
- `completedCompactions` = compaction user message + finished, non-error summary child (`packages/opencode/src/session/compaction.ts:97-113`). Those pairs are **hidden** from the input passed to `select`; the newest summary text becomes `previousSummary` (`:363-371`). The new range is everything else, including the previously retained tail.

### v2 runtime
- Previous checkpoint's `recent` text is fed back as conversation together with the new head, and its `summary` passed as prior summary (`packages/core/src/session/compaction.ts:183-188`); skipped only when there is no head and no prior checkpoint (`:184`). Spec: "Repeated compactions update the previous structured summary with newly compacted messages", then the runner reloads history and executes the pending turn (`specs/v2/session.md:119`).

## Constants
| name | value | path:line |
|---|---|---|
| prior-summary tag | `<prior-summary>` (was `<previous-summary>`) | `packages/core/src/session/compaction.ts:170` |

## Evolution
- 2026-04-22 `574b2c2170` (#23870) "anchored" summary: `<previous-summary>` block, "Update it … preserving still-true details, removing stale details".
- 2026-08-12 `dab2637217` (#42045) for small models like DeepSeek V4 Flash: conversation moved before the prior summary, tag renamed `<prior-summary>`, explicit "discarded after this" + carry-forward + conflict rules → [[summary-template-drops-goals]].

## Quirks / drift
- Prompt permits dropping "only what is finished and no longer needed" — bounded size vs loss, same trade as pi.

Contrast: [[pi--iterative-summary-update|pi]] puts `<previous-summary>` after `<conversation>` in a separate update prompt with 6 merge rules; opencode converged on the same shape and then added a blunt "anything you do not carry is lost" warning for small summarizers.
