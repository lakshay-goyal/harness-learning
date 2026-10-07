---
type: implementation
harness: codex
concept: auto-compaction
commit: 622e9e3696
files: [codex-rs/core/src/session/turn.rs:1434, codex-rs/core/src/session/turn.rs:1303, codex-rs/core/src/session/turn.rs:607, codex-rs/core/src/session/turn.rs:716, codex-rs/core/src/session/turn.rs:1337, codex-rs/core/src/session/turn.rs:1462, codex-rs/core/src/compact.rs:63, codex-rs/core/src/compact.rs:113, codex-rs/core/src/compact.rs:259, codex-rs/core/src/compact.rs:330, codex-rs/core/src/compact.rs:383, codex-rs/core/src/compact.rs:583, codex-rs/core/src/compact.rs:686, codex-rs/core/src/compact_remote_v2.rs:73, codex-rs/core/src/compact_remote_v2_attempt.rs:84, codex-rs/core/src/compact_model_fallback.rs:13, codex-rs/core/src/tasks/compact.rs:53, codex-rs/prompts/templates/compact/prompt.md:1, codex-rs/prompts/templates/compact/summary_prefix.md:1, codex-rs/protocol/src/openai_models.rs:527]
---
[[auto-compaction]] in [[codex]].

## Mechanism
- **Dispatcher** `run_auto_compact` (`codex-rs/core/src/session/turn.rs:1434`): `Feature::TokenBudget` → `compact_token_budget::run_inline_auto_compact_task` (reset, no summary → [[codex--model-requested-context-reset]]); provider `RemoteCompactionSupport::V2` → remote v2; `Unsupported` → local summarizing compaction (`codex-rs/core/src/session/turn.rs:1462-1511`). Manual `/compact` picks the same way (`codex-rs/core/src/tasks/compact.rs:53-92`).
- **Trigger sites**:
  - PreTurn: `run_pre_sampling_compact` before the first sample if `token_status.token_limit_reached` (`codex-rs/core/src/session/turn.rs:1303-1331`).
  - MidTurn "roll over": after each sampling request `should_roll_over = needs_follow_up && (take_new_context_window_request() || token_limit_reached)` (`:607-608`) — only when the model still needs to continue (tool calls pending or queued input). Comment: "as long as compaction works well in getting us way below the token limit, we shouldn't worry about being in an infinite loop" (`:619`) — no loop guard.
  - PostTurn (opt-in): turn finished without follow-up, `model_post_turn_compact_threshold_percent > 0`, not TokenBudget, threshold reached, no pending input, not cancelled (`:716-737`); failure swallowed with `warn!("Post-turn compaction failed; preserving the completed turn")` (`:757`); outputs buffered and committed only on success — "failures must leave both the live history and persisted rollout intact" (`codex-rs/core/src/compact.rs:798-801`); requires non-empty summary ("Post-turn compaction completed without an assistant summary", `:369-379`). Disabled for approval reviewers "to avoid delaying approval completion" (`49e248d4c3`).
  - Model switch: `maybe_run_previous_model_inline_compact` compacts with the PREVIOUS model when its `comp_hash` (compaction-compatibility hash) changed, or when switching to a smaller-window model and usage exceeds the new model's limit/window (`codex-rs/core/src/session/turn.rs:1337-1431`); the new model's step context owns the replacement history (`:1409-1412`); the previous turn's `cyber_access_program` is restored so the server doesn't reject the pair (`:1365-1371`).
- **Threshold**: `auto_compact_token_limit()` = min(catalog/config limit, 90% of context window) (`codex-rs/protocol/src/openai_models.rs:527-539`; catalog ships `null`, `codex-rs/models-manager/models.json:36`); hard cap at `effective_context_window_percent` (95) forces compaction regardless of scope → [[codex--token-estimation]].
- **Scope** `model_auto_compact_token_limit_scope` (`80fdd4688f` 2026-05-19, #22870): `total` (default) counts the whole active context against the clamped limit; `body_after_prefix` counts only tokens after the window's carried prefix (`prefill_input_tokens`, estimated at window start) against the *unclamped* config limit (else catalog limit), the full window still a hard cap (`codex-rs/protocol/src/config_types.rs:49-55`; `codex-rs/core/src/session/context_window.rs:60-85`; `codex-rs/core/src/session/mod.rs:1844-1850`). Model downshift in body scope compacts only when the new window itself is reached (`codex-rs/core/src/session/turn.rs:1385-1398`).
- **Local algorithm** (`codex-rs/core/src/compact.rs:259-426`):
  - Append the summarization prompt (config `compact_prompt` else `SUMMARIZATION_PROMPT`, `:121-125`) as a synthetic *user* message to a clone of the full live history (`:113-130`, `:269-274`); same base instructions as the session unless already present in history (then empty, `:291-300`); stream with the normal model client; last assistant message = summary (`:371-382`).
  - Prompt (`codex-rs/prompts/templates/compact/prompt.md:1-9`, `include_str!` at `codex-rs/prompts/src/compact.rs:1-2`): "You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task." Include: current progress and key decisions; important context, constraints, or user preferences; what remains (clear next steps); critical data, examples, references. "Be concise, structured, and focused on helping the next LLM seamlessly continue the work."
  - Summary item = `format!("{SUMMARY_PREFIX}\n{summary}")` (`:383`); prefix (`codex-rs/prompts/templates/compact/summary_prefix.md:1`): "Another language model started to solve this problem and produced a summary of its thinking process. You also have access to the state of the tools that were used by that language model. Use this to build on the work that has already been done and avoid duplicating work. Here is the summary produced by the other language model, use the information in this summary to assist with your own analysis:" — third-person handoff, not "your earlier conversation".
  - Replacement history = recent real user messages + one `CompactionSummary` user item → [[codex--compaction-cut-point]]; empty summary → "(no summary available)" (`:750-756`) → [[codex--summary-validation]]; all assistant messages, tool calls and outputs dropped.
  - Prior summaries detected by prefix (`is_summary_message`, `:601-603`) and filtered from the kept user messages (`:583-592`); the summarizer still sees them as ordinary history (no merge template).
  - Canonical initial context (developer/env/world state) re-rendered and inserted before the last real user message; if none, before the summary so it stays last (`:80-111`, `:605-671`).
  - One reused `ModelClientSession` so sticky routing / websocket incremental state survive retries (`:282-284`); retries up to provider `stream_max_retries` with `backoff(retries)` and "Reconnecting... n/max" (`:344-357`); overflow of the compaction request itself → drop oldest item and retry (`:330-346`) → [[codex--overflow-recovery]]; aborts immediately on `SessionBudgetExceeded` (`:335-337`).
  - Pre/Post compact hooks can stop compaction → `TurnAborted` (`:189-226`). After success: Warning "Heads up: Long threads and multiple compactions can cause the model to be less accurate. Start a new thread when possible…" (`:419-423`), usage recomputed (`:414`), auto-compact window counter advanced (`:385`).
- **Remote v2** (preferred for OpenAI Responses providers): normal streamed `/responses` request whose input is normalized history + trailing `ResponseItem::CompactionTrigger {}` with full tool list and base instructions — no summarization prompt (`codex-rs/core/src/compact_remote_v2_attempt.rs:84-103`, `:138-144`). Server returns exactly one opaque encrypted `ResponseItem::Compaction`; 0 or >1 → `CodexErr::Fatal` (`codex-rs/core/src/compact_remote_v2.rs:430-492`). Pre-trim: if estimate > window, walk newest→oldest replacing tool outputs with "Output exceeded the available model context and was truncated" until it fits, i128 math (`codex-rs/core/src/compact_remote_history.rs:16-17`, `:68-124`). Replacement = retained items + server item last (`codex-rs/core/src/compact_remote_v2.rs:494-521`) → retention rules in [[codex--compaction-cut-point]]. Retries min(provider, 2) "Compact attempts can run much longer than normal turns" (`:75-77`, `:383-387`). Compaction usage counted against the session budget (`:296-297`).
- **Model fallback**: previous-model compaction rejected (not abort/interrupt/budget; models/programs differ; OpenAI provider with Codex-backend or API-key auth) → retry once with the current model; counter `codex.compaction.model_fallback` (`codex-rs/core/src/compact_model_fallback.rs:13-75`; `codex-rs/core/src/compact_remote_v2.rs:244-283`) → [[compaction-pinned-to-unavailable-model]].
- **Persistence**: `CompactedItem` checkpoint with summary, full `replacement_history`, retained context, window ids, compaction response id, latest token-usage record, resume metadata (`codex-rs/history/src/lib.rs:286-306`); resume reverse-replays to the newest surviving compaction → [[context-projection]].
- **Steering**: input cannot steer a Compact task (`NotSubmittedReason::ActiveTurnNotSteerable`, `codex-rs/core/src/session/turn_input.rs:824-835`); TUI queues follow-ups during `/compact` (`e838645fa2`).

## Constants
| name | value | path:line |
|---|---|---|
| auto-compact threshold | min(config/catalog, 90% of window) | `codex-rs/protocol/src/openai_models.rs:527-539` |
| effective window (hard) | 95 % | `codex-rs/protocol/src/openai_models.rs:391-393` |
| post-turn threshold | 0 = disabled (0..=100) | `codex-rs/core/src/config/mod.rs:3266-3269`, `:4325` |
| `COMPACT_USER_MESSAGE_MAX_TOKENS` | 20_000 | `codex-rs/core/src/compact.rs:63` |
| `RETAINED_MESSAGE_TOKEN_BUDGET` (remote) | 64_000 | `codex-rs/core/src/compact_remote_v2.rs:73` |
| `MAX_REMOTE_COMPACTION_V2_STREAM_RETRIES` | 2 | `codex-rs/core/src/compact_remote_v2.rs:77` |
| local compaction retries | provider `stream_max_retries` | `codex-rs/core/src/compact.rs:344-357` |
| early threshold history | 220_000 → 250_000 tokens (gpt-5-codex) | `b90eeabd74` 2025-09-24 |

## Evolution
- 2025-04-18 `9a948836bf` manual `/compact` in the TS CLI; 2025-07-31 `e2c994e32a` `/compact` in Rust.
- 2025-07-31 `e2c994e32a` (as root `SUMMARY.md`; moved to `6cfee15612:codex-rs/core/src/prompt_for_compact_command.md` 2025-08-08 `6cfee15612` "Moving the compact prompt near where it's used (#2031)", pure rename): "You are a summarization assistant. A conversation follows between a user and a coding-focused AI (Codex)." + fixed template **Objective:** / **User instructions:** / **AI actions / code behavior:** / **Important entities:** / **Open issues / next steps:** / **Summary (concise):** → [[structured-compaction-summary]].
- 2025-09-12 `ea225df22e` "feat: context compaction (#3446)" — first AUTO compaction; in-character memento: "You have exceeded the maximum number of tokens, please stop coding and instead write a short memento message for the next agent." ("If there was a recent update_plan call, repeat its steps verbatim", "List outstanding TODOs with file paths / line numbers", "Flag code that needs more tests", "Record any open bugs, quirks, or setup steps"); `history_bridge.md` rebuilt context as one message ("You were originally given instructions from a user over one or more turns. Here were the user messages: {{ user_messages_text }} … Here is the summary …: {{ summary_text }}"). Body: "Stops the model when the context window become too large / Add a user turn, asking for the model to summarize / Build a bridge that contains all the previous user message + the summary".
- 2025-09-22 `c415827ac2` (#4068) truncate kept user messages → [[summarization-request-overflows]].
- 2025-09-23 `2451b19d13` auto-compaction for gpt-5-codex; 2025-09-24 `b90eeabd74` limit 220k → 250k.
- 2025-10-08 `687a13bbe5` (#4942) "truncate on compact … iteratively".
- 2025-10-20 `049a61bcfc` "Auto compact at ~90% (#5292)": "Users now hit a window exceeded limit and they usually don't know what to do. This starts auto compact at ~90% of the window." (also `effective_context_window_percent`).
- 2025-10-30 `f4f9695978` "feat: compaction prompt configurable (#5959)".
- 2025-10-31 `611e00c862` "feat: compactor 2 (#6027)": memento removed → neutral "CONTEXT CHECKPOINT COMPACTION … handoff summary for another LLM"; `history_bridge.md` deleted, user messages re-inserted as real items → [[domain-biased-summarizer-prompt]].
- 2025-11-14 `0b28e72b66` "Improve compact (#6692)": `summary_prefix.md`; "Allow multiple compaction for long running tasks … Filter out summary messages on the following compaction … We need to address having multiple user messages because it confuses the model … Theoretically, we can end up in infinite compaction loop if the user messages > compaction limit" → [[injected-summary-indistinguishable]].
- 2025-11-18 `838531d3e4` "remote compaction (#6795)" (`/v1/responses/compact`), on by default `cac0a6a29d` same day; API-key users `b3ddd50eee` 2025-12-12.
- 2025-11-21 `b519267d05` account for encrypted reasoning in the trigger → [[estimator-undercounts-context]].
- 2026-01-28 `26590d7927` "Ensure auto-compaction starts after turn started (#10129)" → [[compaction-cancellation-races]].
- 2026-02-06 `ba8b5d9018` "Treat compaction failure as failure state (#10927)" — stop turns instead of continuing.
- 2026-02-11 `40de788c4d` "Clamp auto-compact limit to context window (#11516)" → [[auto-compact-threshold-exceeds-window]].
- 2026-02-20 `bb0ac5be70` "Fix compaction context reinjection and model baselines (#12252)".
- 2026-04-27 `4e05f3053c` "Remove ghost snapshots (#19481)": ghost-commit items no longer carried through history/compaction.
- 2026-04-27 `85c1500569` deferred tools filtered for compaction requests too → [[deferred-tools-lost-on-resume]].
- 2026-05-11 `15e79f3c26` in-turn overflow compaction hardening, reverted same day `69f3183a8e` → [[overflow-compaction-cascade]].
- 2026-06-01 `ba2b67f9cd` prompts moved to `codex-rs/prompts/templates/compact` (text unchanged).
- 2026-06-10 `ba4925b3c2` "[codex] Compact when comp_hash changes (#27520)".
- 2026-07-07 `172ab264bd` "fix: retry rejected previous-model compaction with selected model (#30319)".
- 2026-08-18 `711a5f8b3a`, 2026-09-22 `c117207a6f` drop descendant progress / channel posts from retained history; 2026-08-23 `6677fd827d` budget retained images → [[compaction-loses-modality-or-structure]].
- 2026-08 → 2026-10 ~15 "Preserve X across compaction" fixes → [[compaction-drops-harness-state]].
- 2026-09-08 `35d9e4bc4d` reasoning-effort pin vs compaction; 2026-09-25 `5f3180c793` restore access program → [[compaction-request-shape-mismatch]].
- 2026-09-09 `3dc1e2a584` "Always use streamed remote compaction for supported providers (#44255)" retired the toggle; `1ac689cc7d` "Remove the unused legacy remote compaction implementation (#44273)" deleted the `/responses/compact` v1 client. Bedrock Responses compaction `b9ba969f39` (#36981), `f20b63e85c` (#39825).
- 2026-09-10 `ee93abb690` preserve incoming prompts when pre-turn compaction fails → [[compaction-drops-pending-prompt]].
- 2026-09-18 `49e248d4c3` "Add opt-in compaction after final responses (#46541)".
- 2026-09-25 `bd3d4d1436` (#48115) keep original parts for text-only kept messages.
- **Rejected experiments** (side branches, never merged; `git merge-base --is-ancestor` false): `edd46c6347` / `99b78c9bb2` / `e39e0c4332` 2025-10-30 strict-JSON summary `{intent_user_message, summary}` with `<VERBATIM_REQUEST_START>`…`<VERBATIM_REQUEST_END>`, `<RECENT_USER_CONTEXT_START>`, "Reconstruct the SINGLE ACTIVE TASK THREAD", "RESUME_AT: <next precise action>", "SELF-CHECK BEFORE EMITTING"; `e67726a994` 2025-11-19 (branch tibo-openai-patch-1) revert to the memento wording; `71163530a4` also side-branch. Mainline kept the short free-form prompt and moved verbatim-request preservation into code.

## Quirks
- `SUMMARIZATION_PROMPT` is now only the fallback for non-Responses providers; OpenAI models compact server-side with an opaque summary the client cannot read.
- No loop guard on mid-turn rollover; relies on compaction shrinking far below the limit.
- Transcript doubles as state store → recurring compaction-survival tax.

## Versus pi
- [[pi--auto-compaction]]: fixed headroom (window − 16384), keeps a 20k recent *mixed* tail with cut-point rules, structured iterative summary, append-only entry. codex: 90 % ratio, keeps only user messages (20k) + third-person handoff summary or an opaque server item, re-injects canonical context, compacts across model switches with the old model. → [[compaction-locus]].
