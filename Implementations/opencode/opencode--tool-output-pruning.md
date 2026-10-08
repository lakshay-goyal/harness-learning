---
type: implementation
harness: opencode
concept: tool-output-pruning
commit: ecc4916b5a
files: [packages/opencode/src/session/compaction.ts:28-31, packages/opencode/src/session/compaction.ts:271-317, packages/opencode/src/session/prompt.ts:1338, packages/opencode/src/session/message-v2.ts:302-309, packages/core/src/v1/config/config.ts:154-156, packages/opencode/src/config/config.ts:596-598, specs/v2/session.md:121]
---
[[tool-output-pruning]] in [[opencode]].

## Mechanism

### Legacy runtime — `SessionCompaction.prune`
- Runs after the agent loop ends, forked into the session scope with errors ignored (`packages/opencode/src/session/prompt.ts:1338`).
- Gate: `if (!cfg.compaction?.prune) return` — **off unless explicitly enabled** (`packages/opencode/src/session/compaction.ts:274-275`); `OPENCODE_DISABLE_PRUNE` forces off (`packages/opencode/src/config/config.ts:596-598`); schema text "Enable pruning of old tool outputs (default: false)" (`packages/core/src/v1/config/config.ts:154-156`).
- Walk (`compaction.ts:288-305`): messages newest→oldest; skip until 2 user turns have been passed; stop at a summary assistant or at the first part already `time.compacted`; for each completed tool part (except `skill`) add `Token.estimate(output)`; parts beyond the first 40k tokens are candidates.
- Apply only if candidates total > 20k tokens (`:308-316`): set `part.state.time.compacted = Date.now()` and update the row — output bytes stay in SQLite.
- Rendering: pruned parts become "[Old tool result content cleared]" and lose attachments at request build (`packages/opencode/src/session/message-v2.ts:302-309`) and in the summarizer transcript ([[opencode--transcript-serialization-for-summary]]).

### v2 runtime
- Absent: "Deterministic old tool-result pruning remains a separate follow-up" (`specs/v2/session.md:121`). Provider-executed results "require provider-aware pruning or compaction" (`CONTEXT.md:199`).

## Constants
| name | value | path:line |
|---|---|---|
| `PRUNE_MINIMUM` | 20_000 tokens | `packages/opencode/src/session/compaction.ts:28` |
| `PRUNE_PROTECT` | 40_000 tokens | `packages/opencode/src/session/compaction.ts:29` |
| `PRUNE_PROTECTED_TOOLS` | `["skill"]` | `packages/opencode/src/session/compaction.ts:31` |
| protected recent user turns | 2 | `packages/opencode/src/session/compaction.ts:291` |
| default | off | `packages/opencode/src/session/compaction.ts:275` |

## Evolution
- 2025-09-12 `469dc9095f` "add microcompact" — `time.compacted` + "[Old tool result content cleared]".
- 2025-09-17 `259c722208` "only prune messages from more than 2 turns ago".
- 2025-12-23 `046e351140` native skill tool; skill outputs protected.
- 2025-12-26 `2b054bec95` config checks respect user settings (`prune === false` opt-out).
- 2026-04-09 `9819eb0461` "tweak: disable" and 2026-04-19 `6eddf08244` "flip toolcall prune defaults": opt-out → opt-in; no rationale in either commit.
- 2026-06-03 `9251e5d8c4` docs corrected to "default false".

## Quirks / drift
- Pruning rewrites earlier tool results, so the next request misses the provider cache from the first pruned part; the 20k minimum amortizes that, the default flip suggests it did not pay off (inference).
- Stop-at-first-compacted means a later prune never revisits parts older than the previous prune point.

Contrast: pi has no pruning (considered "OpenCode-style prune" in a plan, not adopted; see [[auto-compaction]]) and relies on summarization alone; opencode shipped pruning on by default for seven months, then made it opt-in.
