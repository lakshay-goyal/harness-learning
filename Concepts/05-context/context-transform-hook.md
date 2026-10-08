---
type: concept
stage: context
tier: must-have
aliases: [transformContext, "context event", context_with_system, emitContext, "context filter", "experimental.chat.messages.transform", "experimental.chat.system.transform"]
harnesses: [pi, opencode]
---
Per-request hook that rewrites the message list sent to the model (prune, inject, hide) without mutating the persisted history.

## Why
- Lets plugins implement pruning, RAG injection, redaction, tool hiding or alternative compaction without forking the loop.
- Must be non-destructive: request-time edits that leak into history are irreversible; pi deep-clones before handing messages to plugins (`structuredClone`).
- When the system prompt and tool declarations live *inside* the message list (see [[transcript-carried-system-prompt]]), a naive "keep last N messages" transform deletes the prompt and tools — pi hit exactly this: requests went out with no tools and Codex emitted raw tool-call text ([[context-handler-drops-system-state]]).

## Design space
- Hook level: app-message level before conversion (pi `transformContext` on `AgentMessage[]`) vs wire level after conversion (pi also has `before_provider_request` payload hook) vs whole-request hook (pi `prepareRequest` may swap context/model/thinking).
- What the plugin sees: conversation only, harness restores prompt/tool state afterwards (pi `context` event) ✔ vs full transcript, plugin owns everything (pi `context_with_system`, runs second; harness only *reports* a dropped leading system message) ✔ — pi offers both, two-phase.
- Chain composition: pi chains several harness-internal projections onto the same hook (extension `context` → hidden-declaration projection → forced-system-prompt projection).
- Failure semantics: handler throw → report and continue with previous messages (pi) vs abort request.
- Persistence: request-only (pi) vs recorded (pi's forced system prompt deliberately *not* recorded, `16292398a`).
- codex: no plugin-level per-request transform found (unverified); built-in non-mutating `for_prompt` normalization on a clone of history (`codex-rs/core/src/context_manager/history.rs:600-615`, `:959-979`); plugins carry no code (`codex-rs/plugin/src/manifest.rs:8-58`).
- Message shape seen by the plugin: storage-level messages with parts, before provider conversion (opencode legacy `experimental.chat.messages.transform`) — call/result adjacency preserved by construction.
- Separate system-array hook with a cache guard: harness re-joins plugin-added system entries to keep ≤ 2 cached blocks (opencode legacy `experimental.chat.system.transform`).
- Same hook applied to the summarizer input so compaction sees what the model sees (opencode `4cb29967f6`).
- None yet in a rewritten runtime (opencode v2 parity table: plugin transforms "missing").

## Implementations
- [[pi--context-transform-hook|pi]] — `transformContext` in the agent loop; coding-agent chains extension `context`/`context_with_system` handlers + two internal projections; plus `prepareRequest` swapping in the canonical session projection.
- [[opencode--context-transform-hook|opencode]] — legacy plugin hooks `experimental.chat.messages.transform` (stored messages, every step and during compaction) and `experimental.chat.system.transform` (system array, re-joined to 2 blocks); v2 has none.

## Failures
- [[context-handler-drops-system-state]]
- [[compaction-request-shape-mismatch]] (opencode: transforms skipped during compaction)
- Cross-group: [[cache-breakpoints-miss-stable-segments]] (06; opencode system hook split the system prompt)

## Related
[[message-conversion-layer]] · [[context-projection]] · [[transcript-carried-system-prompt]] · [[turn-lifecycle-hooks]] · [[extension-event-hooks]] · [[auto-compaction]] · [[deferred-tool-loading]]
