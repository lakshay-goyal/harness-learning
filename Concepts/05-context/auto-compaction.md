---
type: concept
stage: compaction
tier: candidate
aliases: [compaction, "/compact", reserveTokens, keepRecentTokens, shouldCompact, _checkCompaction, _runAutoCompaction, COMPACTION_SUMMARY_PREFIX, CompactionEntry, "compaction.modelOverrides", SessionCompaction, compactIfNeeded, COMPACTION_BUFFER, "compaction.reserved", OPENCODE_DISABLE_AUTOCOMPACT, CompactionPart, "experimental.compaction.autocontinue", "<conversation-checkpoint>"]
harnesses: [pi, opencode]
---
When estimated context exceeds window − reserve, replace older history with an LLM summary while keeping a recent window verbatim; the summary re-enters context as a framed message; users can also trigger it manually with a focus hint.

## Why
- Long agentic sessions exceed any window; without compaction the next request overflows (hard error) or the provider silently truncates.
- Compaction is lossy and invalidates the provider prompt cache, so *when* it fires matters: too early wastes context and cache, too late overflows mid tool-loop (pi: [[threshold-check-misses-post-tool-request]]).
- It is a second LLM "run" inside the session: needs its own failure handling, retries, abort, input queueing, idle tracking (pi: [[compaction-failure-crashes-session]], [[summary-call-not-retried]], [[compaction-cancellation-races]], [[side-phase-input-lost]]).

## Design space
- **Trigger**: ratio of window (e.g. 85–90%, recommended in pi's own plan doc `5daef11b4`) vs **fixed headroom** `window − reserveTokens` ✔ pi (16384; large windows compact at ~98%). Per-model overrides (pi `compaction.modelOverrides`, `46bde88a1`).
- **Check points**: after user turn only (pi until Aug 2026) → **before every provider request incl. post-tool turns** ✔ pi (`56700d42e`) + pre-prompt + post-run. Rejected: proactive *abort mid-turn* when nearing threshold (pi removed `5a9d844f9`).
- **What's kept**: recent token budget verbatim (pi keepRecentTokens 20000, taken from Codex `compact.rs`) vs only summary vs pruning old tool outputs first (OpenCode-style "prune", considered in pi plan, not adopted).
- **Summary placement**: summary as user message with preamble ✔ pi / as system text / as assistant message.
- **Storage**: destructive rewrite vs append-only entry pointing at first kept entry ✔ pi (history never deleted; context rebuilt via [[context-projection]]).
- **Summarizer model**: session model ✔ pi (no cheap-summarizer setting; extensions can choose, e.g. Gemini Flash example) vs dedicated small model.
- **Reserve doubles as output budget**: pi reuses reserveTokens for summary `maxTokens` (0.8×) — couples trigger and summary size.
- **Async vs blocking**: blocking ✔ pi coding-agent; background soft-threshold ✔ pi durable → [[background-compaction]].
- **Manual**: `/compact [focus]` appends "Additional focus:" ✔ pi.
- **Plugin override**: pre-hook may cancel or supply its own summary ✔ pi (`session_before_compact`); add context or replace the prompt, veto the auto-continue turn (opencode legacy `experimental.session.compacting`, `experimental.compaction.autocontinue`).
- **Token source for the trigger**: last response's provider usage (opencode legacy, checked on every `finish-step`) vs pre-request chars/4 estimate of the whole request (opencode v2) vs usage anchor + trailing estimate (pi).
- **Reserve only with a declared input limit**: opencode legacy applies `compaction.reserved` (default min(20k, maxOutput)) only when the model has `limit.input`; otherwise `context − maxOutput`. Unknown window (custom models default context 0) disables threshold compaction entirely (opencode legacy and v2).
- **Compaction as a transcript message**: a user message with a `compaction` part is persisted and processed as the loop's next task; summary is an assistant `summary: true` child (opencode legacy) vs a durable `Compaction.Ended` event projected as one user `<conversation-checkpoint>` (opencode v2).
- **Post-compaction resume**: synthetic "Continue if you have next steps…" user turn only for automatic compaction (opencode legacy) vs replay the pending turn from reloaded history (opencode v2) vs `agent.continue()` (pi).
- **Old tool-output pruning first**: [[tool-output-pruning]] (opencode legacy, off by default since 2026-04).
- **Manual compaction absent**: opencode v2 `session.compact` → `OperationUnavailableError`.

## Implementations
- [[pi--auto-compaction|pi]] — fixed-headroom trigger (16384) checked before every request, post-run and pre-prompt; keep 20000 tokens; structured iterative summary as user message; append-only `compaction` entry with system-prompt snapshot; durable package re-implements as checkpointed task with background threshold.
- [[opencode--auto-compaction|opencode]] — legacy: last-usage ≥ `limit.input − 20k` (or `context − maxOutput`) on every step-finish → compaction user message processed by a hidden `compaction` agent, synthetic continue turn; v2: chars/4 estimate of the full request > `context − max(output, 20k)` → checkpoint event, turn replayed.

## Failures
- [[overflow-compaction-cascade]]
- [[fork-boundary-loss]]
- [[threshold-check-misses-post-tool-request]]
- [[repeated-compaction-drops-kept-messages]]
- [[compaction-includes-abandoned-branches]]
- [[stale-usage-drives-compaction]]
- [[compaction-starved-by-missing-usage]]
- [[empty-compaction-summary]]
- [[compaction-failure-crashes-session]]
- [[summary-call-not-retried]]
- [[compaction-request-shape-mismatch]]
- [[summary-output-budget-misfit]]
- [[pre-prompt-compaction-replays-turn]]
- [[summarization-request-overflows]]
- [[compaction-threshold-underflow]]
- [[post-compaction-transcript-ends-on-assistant]]
- [[overflow-ignores-autocompact-optout]]
- [[reordered-context-misidentifies-latest-turn]] (08-state)
- Cross-group: [[compaction-cancellation-races]], [[side-phase-input-lost]] (01-loop), `fork-boundary-loss` (08-state), `side-request-cache-pollution` (06-caching)

## Tradeoffs
- [[compaction-design]]

## Related
[[compaction-cut-point]] · [[split-turn-summary]] · [[structured-compaction-summary]] · [[iterative-summary-update]] · [[transcript-serialization-for-summary]] · [[summary-validation]] · [[file-op-tracking]] · [[background-compaction]] · [[overflow-recovery]] · [[token-estimation]] · [[context-projection]] · [[context-edit-overlay]] · [[cache-retention-control]] · [[session-handoff]] · [[durable-execution]]
