---
type: implementation
harness: opencode
concept: compaction-cut-point
commit: ecc4916b5a
files: [packages/opencode/src/session/compaction.ts:32-33, packages/opencode/src/session/compaction.ts:115-163, packages/opencode/src/session/compaction.ts:215-269, packages/opencode/src/session/compaction.ts:461-466, packages/opencode/src/session/message-v2.ts:534-585, packages/core/src/session/compaction.ts:137-158]
---
[[compaction-cut-point]] in [[opencode]].

## Mechanism

### Legacy runtime — token-budgeted recent user turns, persisted as `tail_start_id`
- Budget `preserveRecentBudget` = `compaction.preserve_recent_tokens ?? clamp(floor(usable × 0.25), 2_000, 15_000)` (`packages/opencode/src/session/compaction.ts:115-120`) — scales with the window.
- `turns()` = user messages (not compaction parts) with their following assistant/tool messages (`:122-138`).
- `select` (`:223-269`): optional `compaction.tail_turns` caps the number of turns (`≤ 0` → no tail, summarize everything); walk turns newest→oldest, estimating each lazily as `Token.estimate(JSON.stringify(toModelMessages(turn)))` (`:215-221`, "estimate lazily so cost stays proportional to the retained tail"); keep whole turns while within budget.
- First turn that does not fit may be **split**: `splitTurn` tries start indices inside the turn and keeps the first suffix that fits (`:140-163`) — the prefix goes into the summary head, not a separate summary.
- Tail start is persisted on the compaction part (`tail_start_id`, `:461-466`); [[opencode--context-projection|filterCompacted]] restores those messages verbatim after the summary.
- Legal cut = message boundary; tool call and result live in one assistant message's parts, so a message-level cut cannot separate them.

### v2 runtime — serialized text split, no provider messages survive
- Whole projected history serialized to tagged lines; newest lines that fit `keep.tokens` (8k) become `recent`, the rest `head` (`packages/core/src/session/compaction.ts:137-158`).
- `recent` is stored as **text** in the checkpoint (`<recent-context>`), not replayed as provider messages — "Provider-native assistant, reasoning, and tool messages never survive across the boundary, avoiding signature and encrypted-reasoning failures" (`specs/v2/session.md:115`). Cut legality therefore only matters at line granularity.

## Constants
| name | value | path:line |
|---|---|---|
| `MIN_PRESERVE_RECENT_TOKENS` | 2_000 | `packages/opencode/src/session/compaction.ts:32` |
| `MAX_PRESERVE_RECENT_TOKENS` | 15_000 | `packages/opencode/src/session/compaction.ts:33` |
| budget fraction | 0.25 × usable | `packages/opencode/src/session/compaction.ts:118` |
| `tail_turns` default | unset (token budget only) | `packages/core/src/v1/config/config.ts:157-160` |
| v2 `DEFAULT_KEEP_TOKENS` | 8_000 | `packages/core/src/session/compaction.ts:13` |

## Evolution
- 2026-04-10 `6f5a3d30fd` keep recent turns during compaction (tail introduced).
- 2026-04-16 `42771c1db3` budget the retained tail including media.
- 2026-04-20 `c6c56ac2cf` rename `tail_tokens` → `preserve_recent_tokens`.
- 2026-05-05 `811954880e` summary ordered before the retained tail; 2026-05-12 `ca28dd02ec` restore tail turns verbatim instead of folding a serialized tail into the summary → [[repeated-compaction-drops-kept-messages]].
- 2026-05-14 `94564f3588`, 2026-08-07 `db581e47a3` "latest" by time then id after the reorder → [[reordered-context-misidentifies-latest-turn]].
- 2026-01-09 `a5edf3a311`, 2026-04-29 `d71b827d8c` remap parent ids / `tail_start_id` on fork → [[fork-boundary-loss]].
- 2026-08-12 `dab2637217` default `tail_turns: 2` removed, max 8k → 15k, no mid-message character splitting.

## Quirks / drift
- Legacy budgets in chars/4 over `JSON.stringify` of provider messages (JSON overhead inflates the estimate; inference).
- `tail_turns: 0` is the documented way to disable the tail; `preserve_recent_tokens: 0` also yields no tail via `splitTurn`'s `budget <= 0` guard (`:148`).

Contrast: [[pi--compaction-cut-point|pi]] walks projected entries with a fixed 20k keep budget and summarizes a split turn's prefix separately ([[split-turn-summary]]); opencode legacy scales the budget with the window and folds the split prefix into the head, v2 keeps the tail only as serialized text.
