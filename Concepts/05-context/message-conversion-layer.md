---
type: concept
stage: context
tier: candidate
aliases: [AgentMessage, convertToLlm, CustomAgentMessages, "custom roles", bashExecution, compactionSummary, branchSummary]
harnesses: [pi]
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

## Implementations
- [[pi--message-conversion-layer|pi]] — `AgentMessage` union + `convertToLlm` per request; 4 coding-agent custom roles all become user text; sdk wraps it with an image-blocking filter.

## Failures
- [[context-handler-drops-system-state]] — a plugin transform sitting before conversion dropped the prompt/tool system messages.
- [[side-channel-message-splits-tool-pair]] — custom messages appended mid-turn broke call/result adjacency.

## Related
[[context-transform-hook]] · [[cross-provider-handoff]] · [[transcript-replay-repair]] · [[transcript-carried-system-prompt]] · [[out-of-band-message-deferral]] · [[image-normalization]] · [[auto-compaction]] · [[branch-summary]] · [[context-projection]]
