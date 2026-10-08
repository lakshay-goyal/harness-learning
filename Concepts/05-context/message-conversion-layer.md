---
type: concept
stage: context
tier: must-have
aliases: [AgentMessage, convertToLlm, CustomAgentMessages, "custom roles", bashExecution, compactionSummary, branchSummary, toModelMessages, MessageV2, toLLMMessages, SessionMessage]
harnesses: [pi, opencode]
---
Harness keeps its own richer message union (shell runs, plugin notes, summaries, system deltas) and converts it to provider messages only at the request boundary, so storage/UI types never leak to the wire.

## Why
- History must hold things no provider understands: user `!cmd` shell output, plugin-injected notes, compaction/branch summaries, UI-only notices. Without a conversion seam either the provider rejects them or the harness loses them from the log.
- One choke point lets a policy apply to *every* request (e.g. image blocking, `excludeFromContext`) without rewriting history — pi's `blockImages` wrapper is exactly this ([[pi--message-conversion-layer|pi]]).
- Summaries need a framing preamble at render time; storing raw summary + rendering the wrapper keeps the stored text reusable (pi re-feeds stored summaries into the next compaction).

## Design space
- **Single wire type everywhere** (history = provider messages) — simplest, but custom content must be pre-rendered as user text at write time and can't be re-rendered later.
- **App union + converter at request time** — pi: `AgentMessage = Message | CustomAgentMessages[...]` via TypeScript declaration merging, `convertToLlm` per request. ✔ pi
- Converter must be total and non-throwing (pi: exhaustive `never` switch; contract "must not throw").
- Where custom roles land: pi maps all custom roles to `user` messages (shell output, plugin text, summaries); alternative = system messages or tool results.
- Filtering vs placeholder: drop unconvertible messages (pi default converter keeps only system/user/assistant/toolResult) vs replace with text.
- Wrapping converter for policy (pi `blockImages`, checked per request so mid-session toggles apply).
- codex: absent — history *is* the wire type (Responses `ResponseItem` in a harness-metadata envelope, `codex-rs/history/src/lib.rs:210-226`); harness content is pre-rendered at record time into user/developer fragments tagged with `ContentItemKind` so it can still be recognized (`codex-rs/context-fragments/src/fragment.rs:30-64`); only a request-time normalization pass runs on a clone (`codex-rs/core/src/context_manager/history.rs:600-615`) → [[transcript-replay-repair]], [[message-role-layering]].
- **Stored message + typed parts** (text/reasoning/tool state machine/file/step-start/step-finish/patch/compaction/subtask) rendered with placeholders at conversion: compaction part → "What did we do so far?", pruned tool output → "[Old tool result content cleared]", pending tool → "[Tool execution was interrupted]" (opencode legacy, two-stage via AI SDK `UIMessage`).
- **Projected session message variants** lowered to a canonical provider-neutral request, with per-protocol lowering of chronological system updates (opencode v2: `user`, `synthetic`, `system`, `shell`, `assistant`, `compaction`; switch markers dropped).

## Implementations
- [[pi--message-conversion-layer|pi]] — `AgentMessage` union + `convertToLlm` per request; 4 coding-agent custom roles all become user text; sdk wraps it with an image-blocking filter.
- [[opencode--message-conversion-layer|opencode]] — legacy `MessageV2.toModelMessagesEffect` (stored parts → UI messages → ModelMessages, errored turns dropped, interrupted tools answered); v2 `toLLMMessages` over projected `SessionMessage` variants.

## Failures
- [[context-handler-drops-system-state]] — a plugin transform sitting before conversion dropped the prompt/tool system messages.
- [[side-channel-message-splits-tool-pair]] — custom messages appended mid-turn broke call/result adjacency.
- [[post-compaction-transcript-ends-on-assistant]] — opencode: transcript left ending on the summary.

## Related
[[context-transform-hook]] · [[cross-provider-handoff]] · [[transcript-replay-repair]] · [[transcript-carried-system-prompt]] · [[out-of-band-message-deferral]] · [[image-normalization]] · [[auto-compaction]] · [[branch-summary]] · [[context-projection]]
