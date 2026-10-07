---
type: concept
stage: messages
tier: candidate
aliases: [thread title generation, "/rename suggestion", THREAD_TITLE_MODEL, start_temporary_thread, "ThreadSource::Feature(\"thread_title\")", side-prompt, cheap-side-model]
harnesses: [codex]
---
Small auxiliary LLM calls on a cheap model, in a separate temporary thread with schema-constrained output, produce UI metadata (thread titles, rename suggestions) and never enter the main transcript.

## Why
- Titles/labels need language understanding but must not cost main-model tokens, pollute the main transcript, or bust its cache.
- A side prompt is still a prompt: without "Do not answer the request" a small model answers the user's message instead of titling it.
- Results race with the user (manual rename, thread switch) → need cancellation and per-thread attribution.

## Design space
- No generated metadata (title = first user message or a user-set name).
- **Separate temporary thread + cheap model + low reasoning + JSON schema** ✔ codex (titles); same pattern for approval review and memory extraction ✔ codex.
- Fallback when cheap model unavailable: current session model ✔ codex.
- Placement: client (TUI) vs core ✔ codex TUI-only (other clients unverified).
- Cancel on manual rename; attribute result to originating thread ✔ codex.

## Implementations
- [[codex--side-model-metadata-generation|codex]] — `codex-rs/tui/src/app/thread_title.rs`: ≤36-char imperative title, schema `{title, maxLength 36}`, `gpt-5.6-luna` at Low effort for OpenAI+ChatGPT, else current model.

## Failures
- (none as model failures; harness races fixed `a1294e57f1`, `b4c864dd64`)

## Related
[[llm-approval-reviewer]] · [[cross-session-memory]] · [[structured-classifier-api]] · [[cache-stable-prompt-prefix]] · [[delegate-session-runner]]
