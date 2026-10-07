---
type: implementation
harness: opencode
concept: overflow-recovery
commit: ecc4916b5a
files: [packages/opencode/src/session/processor.ts:621-632, packages/opencode/src/session/prompt.ts:1320-1328, packages/opencode/src/session/compaction.ts:340-356, packages/opencode/src/session/compaction.ts:450-459, packages/opencode/src/session/compaction.ts:468-495, packages/opencode/src/session/compaction.ts:527-531, packages/core/src/session/runner/llm.ts:240-298, packages/core/src/session/runner/llm.ts:364-390]
---
[[overflow-recovery]] in [[opencode]].

## Mechanism

### Legacy runtime — compact before the overflowing turn, replay it
- Provider `ContextOverflowError` (incl. HTTP 413) → `needsCompaction` unless `compaction.auto === false`, which now surfaces the error instead (`packages/opencode/src/session/processor.ts:621-632`, `7e09660c3b`).
- Loop creates a compaction with `overflow: !handle.message.finish` (`packages/opencode/src/session/prompt.ts:1320-1328`).
- `process` with `overflow`: find the last real user message before the compaction marker → **held out** as `replay`, summarize only the history before it (`packages/opencode/src/session/compaction.ts:340-356`); if nothing earlier exists, summarize everything.
- After summary: re-create the replay user message with its parts, media file parts replaced by `[Attached <mime>: <filename>]` (`:468-495`). Without a replay candidate, the continue message explains "The previous request exceeded the provider's size limit due to large media attachments…" (`:527-531`).
- If the compaction request itself overflows → `ContextOverflowError` "Conversation history too large to compact…" / "Session too large to compact…", `return "stop"` (`:450-459`): one attempt, no cascade.

### v2 runtime — one compaction, one physical retry, never after durable output
- Overflow-classified provider errors before any assistant output are **held back**, not published (`packages/core/src/session/runner/llm.ts:244-249`).
- After the stream: if `recoverOverflow` is set, nothing was published, and compaction succeeds → `continueAfterOverflowCompaction` (`:291-297`); the retry runs through `runAfterOverflowCompaction`, which has no recovery and dies on a second overflow ("Post-compaction provider attempt cannot recover another overflow", `:364-376`).
- Spec: "A second overflow, unavailable compaction, or overflow after durable output becomes the ordinary terminal failure; recovery never loops or replays partial side effects" (`specs/v2/session.md:121`).

## Constants
| name | value | path:line |
|---|---|---|
| attempts per turn (both runtimes) | 1 | `packages/opencode/src/session/compaction.ts:450-459`; `packages/core/src/session/runner/llm.ts:369-370` |

## Evolution
- 2026-03-02 `be20f865ac` (#14707) recover from 413 via auto-compaction, with replay and media stripping.
- 2026-06-04 `7e09660c3b` respect disabled auto compaction on overflow.
- 2026-06-05 `820c984d47` (#31005) recover v2 context overflow.
- 2026-07-19 `2a097f3af7`, 2026-07-07 `adf178a6b9` overflow regex additions (classifier → [[context-overflow-detection]]).

## Quirks / drift
- v2 `compactAfterOverflow` ignores `compaction.auto` (inference, see [[opencode--auto-compaction]]).
- Legacy replays the user turn *after* the summary, so the oversized turn is never summarized — but if that turn alone exceeds the window the replay overflows again and the second compaction stops the session.

## Failures
[[overflow-compaction-cascade]] · [[completed-response-retried-after-overflow]] · [[overflow-ignores-autocompact-optout]]

Contrast: [[pi--overflow-recovery|pi]] omits the failed attempt via `context_edit`, compacts and `continue()`s with a per-turn latch; opencode legacy holds out the triggering user message and replays it after the summary, v2 holds the error unpublished and retries through a recovery-free path.
