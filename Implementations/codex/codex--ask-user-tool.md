---
type: implementation
harness: codex
concept: ask-user-tool
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/request_user_input_spec.rs:9-141, codex-rs/core/src/tools/handlers/request_user_input.rs:30-105, codex-rs/core/src/tools/spec_plan.rs:1244-1289]
---
[[ask-user-tool]] in [[codex]].

## Mechanism
- Name `REQUEST_USER_INPUT_TOOL_NAME = "request_user_input"` (`codex-rs/core/src/tools/handlers/request_user_input_spec.rs:9`).
- Schema (`request_user_input_spec.rs:16-84`): `questions` — "Questions to show the user. Prefer 1 and do not exceed 3" — each `{id: "Stable identifier for mapping answers (snake_case).", header: "Short header label shown in the UI (12 or fewer chars).", question: "Single-sentence prompt shown to the user.", options}`; options `[{label: "User-facing label (1-5 words).", description: "One short sentence explaining impact/tradeoff if selected."}]` with "Provide 2-3 mutually exclusive choices. Put the recommended option first and suffix its label with "(Recommended)". Do not include an "Other" option in this list; the client will add a free-form "Other" option automatically." (`:35`).
- Validation: "request_user_input requires non-empty options for every question" (`:113`); every question gets `is_other = true` (client-side "Other", `3ba702c5b6`).
- Description templated from the allowed collaboration modes: "Request user input for one to three short questions and wait for the response. This tool is only available in {allowed_modes}." with `no modes` / `X mode` / `X or Y mode` / `modes: a,b` (`:123-141`); called in another mode → "request_user_input is unavailable in {mode} mode" (`:92-104`) — description and runtime check share one policy ([[tool-description-design]], [[plan-mode]]).
- Handler awaits the client response; cancelled → `RespondToModel("request_user_input was cancelled before receiving a response")`; response serialized as JSON text; serialization failure / client channel failure → `Fatal` (ends turn) (`codex-rs/core/src/tools/handlers/request_user_input.rs:90-105`). With `Feature::GuardianApproval` answers are persisted for the reviewer (`:103-105`) — reviewer prompt lists "responses to the `request_user_input` tool" as trusted content (M5b; [[llm-approval-reviewer]]).
- Token-count event deferred until tools resolve so clients don't show progress while the turn is paused on the question (`codex-rs/core/src/session/turn.rs:3167-3173`).
- Gate `config.experimental_request_user_input_enabled`, exposure `ToolExposure::DirectModelOnly` (not on code mode's `tools`) (`codex-rs/core/src/tools/spec_plan.rs:1244-1251`); not parallel-safe → exclusive.
- Async variants, root agent only (`!session_source.is_non_root_agent()`): `RequestUserInputAsyncHandler` when the catalog's `experimental_supported_tools` lists `request_user_input_async` or legacy `send_user_message_async` ("Existing model catalogs still advertise the previous name"), with description/parameters from model-catalog messages; `SendMessageToUserAsyncHandler` with `Feature::SendMessageToUserAsync` or catalog `send_message_to_user_async` (`spec_plan.rs:1253-1289`) → catalog-owned descriptions ([[tool-description-design]]).

## Constants
| name | value | path:line |
|---|---|---|
| questions per call | prefer 1, max 3 | `codex-rs/core/src/tools/handlers/request_user_input_spec.rs:72` |
| options per question | 2-3 (+ client "Other") | `request_user_input_spec.rs:35` |
| header length | ≤12 chars | `request_user_input_spec.rs:49` |
| option label | 1-5 words | `request_user_input_spec.rs:20` |

## Evolution
- 2026-01-19 `57ec3a8277` "Feat: request user input tool (#9472)" (same day `d544adf71a` plan-mode prompt update); 2026-01-20 `0523a259c8` rejected in Execute and Custom modes.
- 2026-01-26 `47aa1f3b6a` "Reject request_user_input outside Plan/Pair"; `3ba702c5b6` `isOther` added (client handles "Other").
- 2026-01-31 `3dd9a37e0b` Plan-mode "Hard interaction rule … Every assistant turn MUST be exactly one of: A) a `request_user_input` tool call…" + "No questions in free text" → "Strongly prefer using the `request_user_input` tool… In rare cases… you may ask it directly" ([[guideline-softening]]).
- 2026-02-03 `998eb8f32b` mode-scoped description ("Tool description and runtime availability checks should be driven by the same centralized mode policy"; tool unavailable in Default) → 2026-02-25 `2f4d6ded1d` re-enabled in Default → 2026-08-29 `a5c581e247` "Clarify question handling in Default collaboration mode".
- 2026-05-29 `1c55bb2702` explicit defaults/bounds in parameter docs.
- 2026-09-02 `2c79ee6dac` structured asynchronous user input requests; `d6350e24be` free-form asynchronous user messages.

## Versus pi
pi has no question tool; its failure [[interactive-tool-in-parallel-batch]] (parallel human-question tools unanswerable) is avoided in codex because the tool is exclusive in the RwLock gate and excluded from code mode.
