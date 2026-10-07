---
type: concept
stage: model-interface
tier: candidate
aliases: [session-title-generation, turn-summary-generation, side call, small model, small_model, getSmallModel, ensureTitle, title agent, title.txt, summary agent]
harnesses: [opencode]
---
Secondary requests to a cheaper model, issued beside the main loop with their own prompt and options. Examples: session title, per-message summary.

## Why
- UI needs (titles, summaries) should not cost main-model tokens or block the turn.
- Main-model options (thinking variant, effort, tools) leak into side calls and break them ([[thinking-config-per-model-drift]]).
- Small models misbehave: they answer the conversation, refuse, think aloud, echo tool narration, drift language, break format ([[small-model-format-noncompliance]], [[title-from-tool-narration]]).

## Design space
- **Model choice**: configured small model → plugin hook → family priority → user's model (opencode) · same model as the turn.
- **Options isolation**: separate `smallOptions` (lowest effort / thinking off), user variant skipped (opencode `554572bc39`) · inherit.
- **Trigger**: once, first real user message (opencode title) · every message (opencode summaries, removed as cost) · on compaction.
- **Scheduling**: forked outside the run's cancel scope (opencode) · awaited inside the turn.
- **Output hygiene**: strip `<think>`, first non-empty line, length cap (opencode) · trust the model.
- **Retries**: own small SDK retry budget (opencode `retries: 2`) · shared session retry.
- pi: partial — only summarization side calls ([[split-turn-summary]], [[branch-summary]]), no title generation.

## Implementations
- [[opencode--auxiliary-model-calls|opencode]] — hidden `title` agent on the provider small model (T = 0.5, `retries: 2`), post-processed to one line ≤100 chars; `summary` agent defined but its caller removed.

## Failures
- [[title-from-tool-narration]]
- [[small-model-format-noncompliance]]
- [[summary-drops-pending-user-request]]
- [[summarizer-continues-conversation]] (05-context)
- [[summarizer-refusal]] (05-context)
- [[side-call-language-drift]] (05-context)
- [[imperative-guideline-over-compliance]] (04-prompting)

## Related
[[model-resolution]] · [[thinking-level-abstraction]] · [[transcript-serialization-for-summary]] · [[split-turn-summary]] · [[abort-propagation]] · [[agent-profiles]]
