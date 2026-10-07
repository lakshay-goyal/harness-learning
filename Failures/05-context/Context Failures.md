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
- [[auto-compact-threshold-exceeds-window]] — codex: configured limit above the window → never compacted; clamp to 90 %.
- [[summary-estimator-payload-mismatch]] — codex: pre-trim estimated different base instructions than were sent.
- [[compaction-drops-pending-prompt]] — codex: failed pre-turn compaction lost the accepted user prompt.
- [[compaction-threshold-underflow]] — catalog output ≥ context made the threshold ≤ 0 → summarize every turn (opencode).
- [[post-compaction-transcript-ends-on-assistant]] — transcript ended on the summary; agent idle until a synthetic continue turn was added (opencode).

## Cut point & range
- [[repeated-compaction-drops-kept-messages]] — second compaction lost the first one's kept messages.
- [[oversized-trailing-tool-results-uncompactable]] — oversized trailing tool results → nothing compacted → overflow.
- [[compaction-loses-modality-or-structure]] — codex: kept messages flattened to text; images not charged to the retained budget.
- [[compaction-drops-harness-state]] — codex: ~15 "Preserve X across compaction" fixes for state living only in history.

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
- [[compaction-request-shape-mismatch]] — summary requests forced reasoning/toolChoice/endpoints the session didn't use; codex: effort pin / access-program pairing.
- [[injected-summary-indistinguishable]] — codex: bare-user-message summary re-summarized as a request; fixed with SUMMARY_PREFIX framing.
- [[side-call-language-drift]] — summaries and titles came back in English for non-English sessions (opencode).

## Overflow recovery
- [[overflow-compaction-cascade]] — overflow → compact → overflow loop (codex: hardening reverted; fail turn, compact next turn).
- [[completed-response-retried-after-overflow]] — successful-but-overflowing response re-run.
- [[overflow-judged-against-wrong-model]] — old model's overflow compacted for a newly selected bigger model.
- [[overflow-ignores-autocompact-optout]] — provider overflow error compacted despite `compaction.auto: false` (opencode legacy fixed; v2 path ungated).
- Cross-group: [[length-stop-recovery]] (01) · [[rate-limit-misread-as-overflow]] · [[overflow-message-not-recognized]] · [[silent-overflow-undetected]] (02)

## Branch summaries
- [[branch-summary-wrong-common-ancestor]] — common ancestor always the root.
- [[branch-summary-records-wrong-source-leaf]] — `fromId` recorded the destination.

## Request assembly
- [[context-handler-drops-system-state]] — plugin context slicing removed prompt + tools; Codex emitted raw tool-call text.
- [[side-channel-message-splits-tool-pair]] — mid-run plugin note landed between tool call and result.

## Cross-session memory
- [[memory-noise-without-noop]] — codex: extractor wrote memories for every rollout; explicit preferred no-op.
- [[memory-overgeneralizes-preferences]] — codex: single-task instructions became always-injected profile rules.
- [[stale-memory-presented-as-current]] — codex: old memory facts stated as verified.

## Images & tool output
- [[image-content-poisoning]] — one bad/oversized image in history rejected every later request (incl. node --watch worker message, `b30a6dd77`).
- [[tool-result-image-routing]] — image-in-tool-result wire shape differs per provider/generation.
- [[truncation-budget-drift-on-replay]] — codex: replay under another model re-truncated tool outputs differently.
- [[prompt-states-stale-harness-limits]] — codex: prompt hard-coded output limits that config later changed.
- [[turn-diff-drops-known-change]] — codex: partially failed patch dropped from the turn diff.
- Cross-group (codex): [[tool-output-bypasses-truncation]] (03) · [[unbounded-payload-in-transcript]] (08) · [[model-switch-replays-unsupported-content]] · [[compaction-pinned-to-unavailable-model]] (02)
- [[tool-output-bypasses-truncation]] — MCP output and appended LSP diagnostics skipped the truncation path (opencode).
- [[spill-failure-reports-successful-side-effect-as-failed]] — v2 spill-write failure turns a completed mutation into "Tool execution failed" (opencode, latent).
- Cross-group: [[bash-output-integrity]] · [[partial-file-read-acted-on]] (03) · [[placeholder-text-misleads-model]] · [[output-token-cap-misbudgeted]] (02) · [[compaction-cancellation-races]] · [[side-phase-input-lost]] · [[queued-messages-stranded-at-run-end]] · [[abandoned-attempts-left-in-context]] (01)

Back to [[Context]].
