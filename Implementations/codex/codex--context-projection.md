---
type: implementation
harness: codex
concept: context-projection
commit: 622e9e3696
files: [codex-rs/core/src/session/rollout_reconstruction.rs:155, codex-rs/core/src/session/rollout_reconstruction.rs:277, codex-rs/core/src/session/rollout_reconstruction.rs:492, codex-rs/core/src/session/rollout_reconstruction.rs:522, codex-rs/core/src/session/mod.rs:4612, codex-rs/core/src/context_manager/history.rs:93, codex-rs/core/src/context_manager/history.rs:983]
---
[[context-projection]] in [[codex]].

Codex projects from the log **only at resume/fork**; during a live session the request context is a mutable in-memory `ContextManager` (`codex-rs/core/src/context_manager/history.rs:93`), not re-derived per request.

## Mechanism
- **Reverse scan to newest surviving compaction**: resume scans rollout items backward, grouping into turn segments bounded by `TurnStarted`, skipping N segments per `ThreadRolledBack{num_turns}` marker, and stops at the newest surviving compaction with `replacement_history`: "A surviving replacement-history compaction is a complete history base. Once we know the newest surviving one, older rollout items do not affect rebuilt history." (`codex-rs/core/src/session/rollout_reconstruction.rs:155-166`, `:277-283`).
- **Forward replay of the suffix** into a fresh `ContextManager`, re-applying the CURRENT model's truncation policy to tool outputs (`codex-rs/core/src/session/rollout_reconstruction.rs:492-551`) — a model switch between sessions changes how old tool output is truncated.
- **Legacy compactions without `replacement_history`**: rebuilt via `build_compacted_history(user messages + summary)` and `reference_context_item` cleared so canonical context is re-injected, accepting "the temporary out-of-distribution prompt shape" (`codex-rs/core/src/session/rollout_reconstruction.rs:522-545`).
- **Context baseline diffing** (live): each turn diffs the new `TurnContextItem` against `reference_context_item` (`TurnReferenceContextItem::{NeverSet,Cleared,Latest}`) and injects only changed context; missing baseline ⇒ full re-injection (`codex-rs/core/src/session/mod.rs:4612-4630`). Rollback trimming a mixed initial-context developer message clears the baseline (`codex-rs/core/src/context_manager/history.rs:983-995`). World-state sections persist as merge patches so resume/fork can keep diffing → [[world-state-diff-injection]].
- **Prompt-time normalization** on the in-memory history: unanswered calls get synthetic "aborted" outputs (`codex-rs/core/src/context_manager/normalize.rs:52-66`) → [[transcript-replay-repair]].
- **Token usage** restored from the compaction's stored usage record instead of scanning far back (`codex-rs/history/src/lib.rs:286-306`).

## Evolution
- 2026-01-06 `8b7ec31ba7` rollback markers (replayed even after API removal `3052bbcf8c` 2026-09-11).
- 2026-06-10 `ba4925b3c2` (#27520) compaction-compatibility hash `comp_hash` persisted in TurnContext.
- 2026-06-22 `3b32d861c5` (#29249) environment context → world state; 2026-06-24 `3e51b46eba`/`fa036d39aa` persist world state.

## Quirks
- Projection cost is paid once per resume, not per request — avoids [[per-request-projection-rescans-log]] but means two sources of truth (memory vs rollout) exist during a session, reconciled only by flush barriers.
- Harness state dropped by compaction is a known failure class → [[compaction-drops-harness-state]].

## Versus pi
- [[pi--context-projection]]: pi rebuilds request messages from the log on every request (leaf→root walk, latest compaction, `context_edit` overlay); codex keeps a live mutable history and only projects on resume/fork; no edit overlay ([[context-edit-overlay]]).
