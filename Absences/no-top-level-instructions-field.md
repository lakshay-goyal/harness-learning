---
type: absence
harnesses: [codex]
---
# no-top-level-instructions-field

The Responses `instructions` field is no longer sent; the base prompt is a leading developer input message with a stable id.

**What's missing**
- `ResponsesApiRequest` has no `instructions` field (`codex-rs/core/src/client.rs:996-1011`); base instructions = developer message id `msg_<uuidv5(thread_id, text)>`, content kind `model.base_instructions` (`codex-rs/core/src/client.rs:931-940`; `codex-rs/core/src/context/base_instructions.rs`).

**Evidence of decision**
- `c9253c4977` 2026-10-05 "Send base instructions as Responses input messages (#51156)".

**Implication**
- Retries/resume keep identical prefix bytes; changed instructions force a full request without `previous_response_id` ([[cache-stable-prompt-prefix]], [[message-role-layering]], [[transcript-carried-system-prompt]]).

Related: [[message-role-layering]] · [[transcript-carried-system-prompt]] · [[cache-stable-prompt-prefix]] · [[Absences]]
