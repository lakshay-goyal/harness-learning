---
type: concept
stage: model-interface
tier: must-have
aliases: [session-title-generation, turn-summary-generation, side call, small model, small_model, getSmallModel, ensureTitle, title agent, title.txt, summary agent, thread title generation, "/rename suggestion", THREAD_TITLE_MODEL, start_temporary_thread, "ThreadSource::Feature(\"thread_title\")", side-prompt, cheap-side-model, side-model-metadata-generation]
harnesses: [opencode, codex]
---
Secondary requests to a cheaper model, issued beside the main loop with their own prompt and options. Examples: session title, per-message summary.

## Why
- UI needs (titles, summaries) should not cost main-model tokens or block the turn.
- Main-model options (thinking variant, effort, tools) leak into side calls and break them ([[thinking-config-per-model-drift]]).
- Small models misbehave: they answer the conversation, refuse, think aloud, echo tool narration, drift language, break format ([[small-model-format-noncompliance]], [[title-from-tool-narration]]).
- Titles/labels need language understanding but must not cost main-model tokens, pollute the main transcript, or bust its cache.
- A side prompt is still a prompt: without "Do not answer the request" a small model answers the user's message instead of titling it.
- Results race with the user (manual rename, thread switch) → need cancellation and per-thread attribution.

## Design space
- **Model choice**: configured small model → plugin hook → family priority → user's model (opencode) · same model as the turn.
- **Options isolation**: separate `smallOptions` (lowest effort / thinking off), user variant skipped (opencode `554572bc39`) · inherit.
- **Trigger**: once, first real user message (opencode title) · every message (opencode summaries, removed as cost) · on compaction.
- **Scheduling**: forked outside the run's cancel scope (opencode) · awaited inside the turn.
- **Output hygiene**: strip `<think>`, first non-empty line, length cap (opencode) · trust the model.
- **Retries**: own small SDK retry budget (opencode `retries: 2`) · shared session retry.
- pi: partial — only summarization side calls ([[split-turn-summary]], [[branch-summary]]), no title generation.
- No generated metadata (title = first user message or a user-set name).
- **Separate temporary thread + cheap model + low reasoning + JSON schema** ✔ codex (titles); same pattern for approval review and memory extraction ✔ codex.
- Fallback when cheap model unavailable: current session model ✔ codex.
- Placement: client (TUI) vs core ✔ codex TUI-only (other clients unverified).
- Cancel on manual rename; attribute result to originating thread ✔ codex.
- (folded from `side-model-metadata-generation`, codex framing) Small auxiliary LLM calls on a cheap model, in a separate temporary thread with schema-constrained output, produce UI metadata (thread titles, rename suggestions) and never enter the main transcript.

## Implementations
- [[opencode--auxiliary-model-calls|opencode]] — hidden `title` agent on the provider small model (T = 0.5, `retries: 2`), post-processed to one line ≤100 chars; `summary` agent defined but its caller removed.
- [[codex--auxiliary-model-calls|codex]] — `codex-rs/tui/src/app/thread_title.rs`: ≤36-char imperative title, schema `{title, maxLength 36}`, `gpt-5.6-luna` at Low effort for OpenAI+ChatGPT, else current model.

## Failures
- [[title-from-tool-narration]]
- [[small-model-format-noncompliance]]
- [[summary-drops-pending-user-request]]
- [[summarizer-continues-conversation]] (05-context)
- [[summarizer-refusal]] (05-context)
- [[side-call-language-drift]] (05-context)
- [[imperative-guideline-over-compliance]] (04-prompting)
- (none as model failures; harness races fixed `a1294e57f1`, `b4c864dd64`)

## Related
[[model-resolution]] · [[thinking-level-abstraction]] · [[transcript-serialization-for-summary]] · [[split-turn-summary]] · [[abort-propagation]] · [[agent-profiles]]
[[llm-approval-reviewer]] · [[cross-session-memory]] · [[structured-classifier-api]] · [[cache-stable-prompt-prefix]] · [[delegate-session-runner]]
