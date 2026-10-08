---
type: implementation
harness: pi
concept: background-compaction
commit: b30a6dd77
files: [packages/durable/src/harness/generation.ts:302, packages/durable/src/harness/generation.ts:153, packages/durable/src/harness/compaction.ts:234, packages/durable/src/harness/compaction.ts:407, packages/durable/src/harness/agent.ts:26, packages/durable/docs/spec.md:2263]
---
[[background-compaction]] in [[pi]] (durable package only; the stable coding-agent compacts blocking — see [[pi--auto-compaction]]).

## Mechanism
- **Threshold selection** `thresholdCompaction(view, planned, window, policy)` (`packages/durable/src/harness/generation.ts:302-323`): tokens = `estimateContext(view, plannedSystemEntries)`; `blocking = window − reserveTokens`; `background = blocking − backgroundTokens`; returns `"blocking"` above blocking, `"background"` above background (only if `backgroundTokens > 0`), and only when `selectCut` finds a cut.
- **Blocking**: generation `prepare` commits a compaction **owned by the generation** (`reason:"threshold"`), waits with `allSettled`, checkpoint `{phase:"prepare", attempt, compacted}`, then re-prepares and sends regardless of outcome (`packages/durable/src/harness/generation.ts:153-163`; `packages/durable/docs/spec.md:3506-3512`).
- **Background**: in the same commit that moves to `request`, if no compaction status exists in `pi.live`, create a **conversation-owned** compaction; generation does not wait (`packages/durable/docs/spec.md:3513-3517`). `createCompaction` marks `background = owner === undefined && reason !== "manual"` (`packages/durable/src/harness/compaction.ts:234-248`).
- **Placement** `placeSummary` (`packages/durable/src/harness/compaction.ts:407-436`): blocking → `tx.appendEntry` directly; conversation-owned → `admitSubmission({type:"write", requestId:"compaction:<taskId>", entry})`, "placed at once when idle, otherwise at the next boundary, or settled `stale`". Comment: "nothing else may append to a busy conversation, so every non-blocking summary goes through admission."
- **Ordering by cut** (`packages/durable/docs/spec.md:2263-2280`): summaries are *head writes*; a head write whose target is older than the active range start is `stale`. Example: background B cuts at 70 while blocking A cuts at 150 and lands first → B settles stale; had A cut at 60, B is placed after it and still covers everything.
- Task state survives crashes (select/summarize/retry checkpoints) → [[durable-execution]].

## Constants
| name | value | path:line |
|---|---|---|
| `backgroundTokens` | 32768 (background starts ~49k below window with default reserve) | `packages/durable/src/harness/agent.ts:26-31` |
| `reserveTokens` / `keepRecentTokens` | 16384 / 20000 | `packages/durable/src/harness/agent.ts:26-31` |

## Evolution
- 2026-09-30 `ed0d6b91b` "feat(durable): compaction and overflow (Package 20)"; 2026-10-01 `b56702ad3` (Package 21) per-conversation agents/policies.

## Evidence commits
`ed0d6b91b` `b56702ad3`

## Quirks
- Spec-listed footguns (`packages/durable/docs/spec.md:4615-4629`): every background compaction pays a summarization request even if it ends `stale`; an application edit placed while a compaction summarizes is lost when the summary cuts past its target; an application head (forget 10-84) is undone by a later-placed summary that still describes 10-84; a queued summary's user message carries the time compaction *finished*, so timestamp-based usage staleness (pi-ai `estimateContextTokens`) can misjudge — the harness estimate uses entry order.
- Coding-agent has no equivalent; whether stable pi will migrate onto durable is unstated (open question).

## Failures
(none mined)
