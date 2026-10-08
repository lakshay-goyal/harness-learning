---
type: implementation
harness: opencode
concept: context-projection
commit: ecc4916b5a
files: [packages/opencode/src/session/message-v2.ts:482-503, packages/opencode/src/session/message-v2.ts:534-617, packages/core/src/session/history.ts:24-53, packages/core/src/session/runner/llm.ts:197-201, CONTEXT.md:11-13, CONTEXT.md:178-179]
---
[[context-projection]] in [[opencode]].

## Mechanism

### Legacy runtime — `filterCompacted` over the SQLite message stream
- `MessageV2.stream` pages newest-first in pages of 50 (`packages/opencode/src/session/message-v2.ts:482-503`); the loop re-reads it at the top of every step, so the log (not memory) is the source of truth.
- `filterCompacted` (`packages/opencode/src/session/message-v2.ts:534-585`): walk newest→oldest, record parents of completed summaries; stop at the newest completed compaction user message — or keep walking back to its `tail_start_id`; reverse. If a tail exists, **reorder** to `[compaction user, summary, …retained tail…, …post-summary messages…]` (`packages/opencode/src/session/message-v2.ts:577-583`).
- Because the projected array is no longer chronological, `latest()` picks the newest user/assistant/finished by `time.created`, then id as tie-breaker (`packages/opencode/src/session/message-v2.ts:591-617`); comment: "IDs are only a deterministic tie-breaker because imported messages do not necessarily have monotonic IDs."
- Pruned tool outputs and error/abort filtering are applied at conversion ([[opencode--message-conversion-layer]]), not in the projection.

### v2 runtime — SQL projection by sequence
- `SessionHistory.entriesForRunner(db, sessionID, baselineSeq)`: rows from `session_message` with `seq >= latest compaction seq`, plus `system` rows only if `seq > baseline_seq`, ordered by `seq` (`packages/core/src/session/history.ts:24-53`) — compaction cutoff and Context Epoch cutoff in one query.
- Session History (_Avoid_: Session Context) = "projected chronological conversation selected for a provider turn after applying the active compaction and Context Epoch cutoffs" (`CONTEXT.md:11-13`); the baseline system context is separate request state, and `sessions.context` returns only the message projection (`CONTEXT.md:178-179`).
- Rebuilt per provider turn (`packages/core/src/session/runner/llm.ts:197-201`).

## Constants
| name | value | path:line |
|---|---|---|
| legacy page size | 50 | `packages/opencode/src/session/message-v2.ts:483` |

## Evolution
- 2026-05-05 `811954880e` summary ordered before retained tail; 2026-05-12 `ca28dd02ec` tail restored verbatim.
- 2026-05-14 `94564f3588` double auto-compaction from the reorder (array-position "latest") → max id; 2026-08-07 `db581e47a3` time then id → [[reordered-context-misidentifies-latest-turn]].
- 2026-06-04 `1af8dafd3e` v2 history with epoch cutoff.

## Quirks / drift
- Legacy loads the whole post-compaction history every step (paged, but fully materialized); long uncompacted sessions pay O(n) per step (inference).

Contrast: [[pi--context-projection|pi]] walks the session tree leaf→root with a `context_edit` overlay; opencode legacy reorders a flat list around `tail_start_id`, v2 selects by event sequence with compaction and epoch cutoffs in SQL.
