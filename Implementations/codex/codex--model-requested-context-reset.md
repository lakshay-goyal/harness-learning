---
type: implementation
harness: codex
concept: model-requested-context-reset
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/new_context_window_spec.rs:6, codex-rs/core/src/tools/handlers/new_context_window.rs:13, codex-rs/core/src/tools/handlers/get_context_remaining_spec.rs:8, codex-rs/core/src/session/mod.rs:4541, codex-rs/core/src/session/token_budget.rs:1, codex-rs/core/src/session/token_budget.rs:161, codex-rs/core/src/session/context_window.rs:97, codex-rs/core/src/compact_token_budget.rs:1, codex-rs/ext/history-notes/src/tools.rs:27, codex-rs/ext/history-notes/src/backend.rs:15, codex-rs/core/src/tools/spec_plan.rs:1295]
---
[[model-requested-context-reset]] in [[codex]].

## Mechanism
- **Gate**: `Feature::TokenBudget` (under development, off; `codex-rs/features/src/lib.rs:1836`; tools planned at `codex-rs/core/src/tools/spec_plan.rs:1295-1298`). Experimental context management only for ChatGPT auth on Plus/Pro/ProLite/ProMax plans (`codex-rs/core/src/session/token_budget.rs:1-55`, table `:235-241`).
- **Tools**: `get_context_remaining` → `{tokens_left: int|null}` (`codex-rs/core/src/tools/handlers/get_context_remaining_spec.rs:8-33`); `new_context`: "Start a new context window. Does not clear, reset, or otherwise affect environment state." returning "A new context window will start without summarizing conversation history." (`codex-rs/core/src/tools/handlers/new_context_window_spec.rs:6-17`, `codex-rs/core/src/tools/handlers/new_context_window.rs:13-14`, `:38-44`). The call sets a flag consumed by the mid-turn rollover check `should_roll_over = needs_follow_up && (take_new_context_window_request() || token_limit_reached)` (`codex-rs/core/src/session/turn.rs:607-608`, `codex-rs/core/src/session/mod.rs:4541-4549`).
- **Reset**: `start_new_context_window` replaces history with freshly built initial context (+ retained client developer messages within 64k) — no summary, no user messages (`codex-rs/core/src/session/mod.rs:4551-4605`); modeled as a compaction lifecycle so hooks and `ContextCompaction` items still fire (`codex-rs/core/src/compact_token_budget.rs:1-50`). Manual `/compact` with the feature on routes to `compact_token_budget::run_manual_compact_task` (`codex-rs/core/src/tasks/compact.rs:53-60`); auto dispatch `codex-rs/core/src/session/turn.rs:1462-1511`.
- **Reminder**: `token_budget::maybe_record` after every sampling request with `base_window_tokens_remaining` (`codex-rs/core/src/session/turn.rs:579-586`); when remaining ≤ `reminder_threshold_tokens` a one-shot `TokenBudgetReminder` (`state.claim_token_budget_reminder()`); when remaining hits 0, no rollover happened and fallback is allowed, one `AutoCompactFallbackPrompt` — a last chance to save state before the forced reset (`codex-rs/core/src/session/token_budget.rs:160-228`). The auto-compact limit is raised by `fallback_buffer_tokens` only when that prompt exists (`codex-rs/core/src/session/context_window.rs:97-103`).
- **Model-owned defaults**: thresholds/templates from catalog `model_messages.token_budget` unless the user configured them (`codex-rs/core/src/session/token_budget.rs:80-150`). Context-window guidance is a world-state section with replacement/removal notices "This context-window guidance replaces all previously provided context-window guidance." / "The previously provided context-window guidance no longer applies." (`codex-rs/core/src/context/world_state/context_window_guidance.rs:8-10`) → [[codex--world-state-diff-injection]].
- **Continuity** (`codex-rs/ext/history-notes`): backend-hosted tools `history.{list_windows,list_items,read_item,search_contents}` and `notes.{list_files_by_prefix,read_file,search_contents,append_to_file,write_file}` hitting `alpha/history/v2/*` and `alpha/notes/v2/*` (`codex-rs/ext/history-notes/src/tools.rs:45-97`). Descriptions: "private model-only state. Use it silently … Never disclose or describe the tool"; note files ≤1,000,000 UTF-8 bytes (`codex-rs/ext/history-notes/src/tools.rs:27-28`). Encrypted tool-arguments header, 35 s backend timeout (`codex-rs/ext/history-notes/src/backend.rs:15-17`).
- **Opt-outs**: token-budget sessions skip post-turn summarizing compaction (`codex-rs/core/src/session/turn.rs:717-719`) and guardian overflow compaction — "token-budget resets do not summarize", must fail closed (`codex-rs/core/src/session/turn.rs:336-342`, `:740-745`, `:763`).

## Constants
| name | value | path:line |
|---|---|---|
| reminder threshold / template | catalog `model_messages.token_budget` defaults (`{n_remaining}`) | `codex-rs/core/src/session/token_budget.rs:81-120` |
| retained client developer messages on reset | 64k tokens | `codex-rs/core/src/session/mod.rs:4551-4605` |
| `HISTORY_NOTES_BACKEND_TIMEOUT` | 35 s | `codex-rs/ext/history-notes/src/backend.rs:15` |
| note file cap | 1,000,000 bytes | `codex-rs/ext/history-notes/src/tools.rs:28` |

## Evolution
- 2026-06-10 `87ab01834a` `new_context` tool: "When the model decides the current window is no longer useful, it needs a way to ask Codex to start over with a fresh context window without spending tokens on a compaction summary."; same day `dac5f07403` remaining-tokens tool.
- 2026-08-21 `daa48072f4` model-callable `history` and `notes` tools "for token-budget sessions… to recover prior conversation context".
- 2026-08-31 `d58d0e5841` "Allow models to enable token budgeting by default".

## Quirks
- Separate from the multi-agent *rollout* budget (weighted ledger shared by a thread tree, `SessionBudgetExceeded`) → [[session-token-budget]].
- The reset is a "compaction" in lifecycle terms only; UI/hooks see a compaction with no summary.

## Versus pi
- pi has only harness-triggered summarizing compaction ([[pi--auto-compaction]]); no remaining-budget signal to the model and no model-initiated reset.
