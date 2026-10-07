---
type: concept
stage: caching
tier: must-have
aliases: [SystemMessage.sections, toolsAdded, toolsRemoved, pi.system, mid-conversation system messages, diffSystemPromptSections, tool_addition, tool_removal, System Context, Context Source, Context Epoch, Context Snapshot, Mid-Conversation System Message, Safe Provider-Turn Boundary, Unavailable Context, unavailable-context-source, SessionContextEpoch, SystemContext.reconcile, "<system-update>", wrapSystemUpdate, session_context_epoch, AdditionalTools, IncrementalTools, use_responses_lite, NAMESPACE_UPDATE_HINT]
harnesses: [pi, opencode, codex]
---
The system prompt (as named sections) and tool declarations live in the transcript as system messages; later changes are appended as deltas (section patches, tool add/remove) instead of rewriting the leading prompt, and are replayed to rebuild current state.

## Why
- Rewriting the head on every prompt/tool change invalidates the provider cache prefix ([[late-tool-change-rewrites-cache]]).
- Silent rewrites make resumed/branched sessions run under different instructions than they were recorded with; transcript deltas make prompt history auditable and restorable.
- Compaction and plugins must not lose prompt/tool state that lives in the transcript ([[context-handler-drops-system-state]]).

## Design space
- Prompt as out-of-band config, rebuilt every request (most harnesses; pi until 2026-09).
- **Leading system message + appended deltas; collapse to one head for models without mid-conversation system support** (**pi chose**, `9e05370b2`).
- Native tool deltas (`tool_addition`/`tool_removal`, `additional_tools`) vs resending full tool list.
- Positional deltas with re-baseline after a head cut (**pi-durable**).
- Forced full prompt: record vs project at request time (**pi**: project, don't record).
- **Instructions as an input item** (codex)
  - The base prompt goes as a `developer` message at `input[0]` with a stable UUIDv5 id instead of the `instructions` field. It is fixed per window and emitted only when the section is absent. ✔ codex (`c9253c4977`)
- **Native tool items with merge semantics stated to the model** (Responses Lite `AdditionalTools` plus incremental updates: "Previously declared tools remain available… unless explicitly marked unavailable"). ✔ codex (`codex-rs/core/src/context/world_state/top_level_tools.rs:21-23`)
- **Typed per-section diffs** for everything else ([[world-state-diff-injection]]). ✔ codex
- **Typed sources instead of text sections**: each Context Source = stable key + JSON codec + baseline/update/removal renderers; change detection by codec equivalence, not text diff; several changed sources combine into one update message (**opencode v2**).
- **Baseline kept out of the transcript**: the immutable Baseline System Context lives in a per-session epoch row reused verbatim; only the chronological updates are transcript messages; a completed compaction starts a new epoch and drops older updates from model history (**opencode v2**).
- **Unavailable vs absent**: a source that cannot be observed keeps its last admitted value (stale-while-revalidate) and blocks only first/replacement baselines; successful absence emits removal text (**opencode v2**; folded slug `unavailable-context-source`).
- **Lazy admission at safe turn boundaries**, after promoted input and settled tool results; never wakes an idle session; admitted updates survive a failed attempt (**opencode v2**).
- **Native mid-conversation `role: system` per model**: only Anthropic `claude-opus-4-8`; every other route gets a visible, XML-escaped `<system-update>` user-text wrapper (**opencode v2**) vs collapse to one head (pi).

## Implementations
- [[pi--transcript-carried-system-prompt|pi]] — `SystemMessage{content, sections, toolsAdded, toolsRemoved}`; `declareToolChanges`; Anthropic inline tools; compaction snapshots `systemMessage`.
- [[codex--transcript-carried-system-prompt|codex]] — partial match. Base instructions as a stable-id developer input item; Lite `AdditionalTools` + incremental tool catalog; AGENTS.md replacement notices; `ConfigurationUpdate` items.
- [[opencode--transcript-carried-system-prompt|opencode]] — v2 only: System Context of keyed Context Sources, durable Context Epoch baseline + Context Snapshot, `ContextUpdated` Mid-Conversation System Messages lowered natively for Opus 4.8 and as `<system-update>` elsewhere; legacy rebuilds the prompt every step.

## Failures
- [[late-tool-change-rewrites-cache]]
- [[forced-system-prompt-applied-as-late-update]]
- [[mcp-startup-blocks-and-description-churn]]
- [[context-handler-drops-system-state]]
- [[tool-loadout-stale-within-run]]
- [[stale-context-files-mid-session]]
- [[permission-context-reinjected-repeatedly]]
- [[strict-schema-rejects-legacy-records]] (08-state; opencode snapshot decode)
- [[stale-runner-recreates-context-after-move]] (08-state)
- [[side-channel-message-splits-tool-pair]] (05-context; opencode guard)

## Related
[[cache-stable-prompt-prefix]] · [[xml-prompt-boundaries]] · [[system-prompt-override]] · [[dynamic-tool-guidelines]] · [[context-projection]] · [[auto-compaction]] · [[signed-reasoning-replay]] · [[durable-execution]] · [[world-state-diff-injection]] · [[cache-preserving-config-update]] · [[prompt-cache-strategy]]

## Tradeoffs
- [[prompt-cache-strategy]]
