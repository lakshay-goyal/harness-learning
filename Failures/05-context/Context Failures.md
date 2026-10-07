---
type: group
group: 05-context
---
Failures whose primary concept is in [[Context]].

## Compaction trigger & accounting
- [[threshold-check-misses-post-tool-request]] — big tool result sent to the provider before compaction ran → overflow mid tool-loop.
- [[stale-usage-drives-compaction]] — kept messages' pre-compaction usage re-triggered compaction immediately.
- [[compaction-starved-by-missing-usage]] — 529 storms / usage-less providers never compacted.
- [[estimator-undercounts-context]] — custom messages, images, optimistic chars/token under-counted context.
- [[compaction-includes-abandoned-branches]] — compaction summarized entries from abandoned tree branches.
- [[pre-prompt-compaction-replays-turn]] — pre-prompt check called `continue()` and replayed the old turn.
- [[compaction-failure-crashes-session]] — quota error in summary call crashed the UI.
- [[summary-call-not-retried]] — one transient stream drop failed the whole compaction.
- [[parallel-side-requests-single-slot-provider]] — parallel split-turn summaries got 429 from single-slot local providers.

## Cut point & range
- [[repeated-compaction-drops-kept-messages]] — second compaction lost the first one's kept messages.
- [[oversized-trailing-tool-results-uncompactable]] — oversized trailing tool results → nothing compacted → overflow.

## Summary prompt & output
- [[summarizer-continues-conversation]] — summarizer answered/continued the chat instead of summarizing.
- [[summarizer-emits-tool-calls]] — summarizer emitted tool calls; `toolChoice:"none"` broke gateways.
- [[summarizer-refusal]] — Fable refused "PREFIX/SUFFIX" split-turn prompt.
- [[domain-biased-summarizer-prompt]] — "AI coding assistant" wording biased non-coding agents.
- [[summary-template-drops-goals]] — "1-2 sentences" Goal dropped goals of multi-task sessions.
- [[summarization-request-overflows]] — summary request itself overflowed (full tool outputs serialized).
- [[truncated-summary-persisted]] — length-stopped partial summary saved as checkpoint.
- [[empty-compaction-summary]] — empty compactions announced/persisted.
- [[summary-output-budget-misfit]] — summary max_tokens above model max; branch cap 2048 eaten by reasoning.
- [[compaction-request-shape-mismatch]] — summary requests forced reasoning/toolChoice/endpoints the session didn't use.

## Overflow recovery
- [[overflow-compaction-cascade]] — overflow → compact → overflow loop.
- [[completed-response-retried-after-overflow]] — successful-but-overflowing response re-run.
- [[overflow-judged-against-wrong-model]] — old model's overflow compacted for a newly selected bigger model.
- Cross-group: [[length-stop-recovery]] (01) · [[rate-limit-misread-as-overflow]] · [[overflow-message-not-recognized]] · [[silent-overflow-undetected]] (02)

## Branch summaries
- [[branch-summary-wrong-common-ancestor]] — common ancestor always the root.
- [[branch-summary-records-wrong-source-leaf]] — `fromId` recorded the destination.

## Request assembly
- [[context-handler-drops-system-state]] — plugin context slicing removed prompt + tools; Codex emitted raw tool-call text.
- [[side-channel-message-splits-tool-pair]] — mid-run plugin note landed between tool call and result.

## Images & tool output
- [[image-content-poisoning]] — one bad/oversized image in history rejected every later request (incl. node --watch worker message, `b30a6dd77`).
- [[tool-result-image-routing]] — image-in-tool-result wire shape differs per provider/generation.
- Cross-group: [[bash-output-integrity]] · [[partial-file-read-acted-on]] (03) · [[placeholder-text-misleads-model]] · [[output-token-cap-misbudgeted]] (02) · [[compaction-cancellation-races]] · [[side-phase-input-lost]] · [[queued-messages-stranded-at-run-end]] · [[abandoned-attempts-left-in-context]] (01)

Back to [[Context]].
