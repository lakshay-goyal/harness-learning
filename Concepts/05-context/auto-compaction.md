---
type: concept
stage: compaction
tier: candidate
aliases: [compaction, "/compact", reserveTokens, keepRecentTokens, shouldCompact, _checkCompaction, _runAutoCompaction, COMPACTION_SUMMARY_PREFIX, CompactionEntry, "compaction.modelOverrides", run_auto_compact, run_inline_auto_compact_task, CompactionPhase, CompactionReason, SUMMARY_PREFIX, "CONTEXT CHECKPOINT COMPACTION", compact_prompt, auto_compact_token_limit, RemoteCompactionSupport, ResponseItem::CompactionTrigger, compact_remote_v2, comp_hash, model_post_turn_compact_threshold_percent]
harnesses: [pi, codex]
---
When estimated context exceeds window − reserve, replace older history with an LLM summary while keeping a recent window verbatim; the summary re-enters context as a framed message; users can also trigger it manually with a focus hint.

## Why
- Long agentic sessions exceed any window; without compaction the next request overflows (hard error) or the provider silently truncates.
- Compaction is lossy and invalidates the provider prompt cache, so *when* it fires matters: too early wastes context and cache, too late overflows mid tool-loop (pi: [[threshold-check-misses-post-tool-request]]).
- It is a second LLM "run" inside the session: needs its own failure handling, retries, abort, input queueing, idle tracking (pi: [[compaction-failure-crashes-session]], [[summary-call-not-retried]], [[compaction-cancellation-races]], [[side-phase-input-lost]]).

## Design space
- **Trigger**: ratio of window (e.g. 85–90%, recommended in pi's own plan doc `5daef11b4`; **✔ codex** min(catalog/config limit, 90% of window) + hard cap at 95% effective window, `049a61bcfc`) vs **fixed headroom** `window − reserveTokens` ✔ pi (16384; large windows compact at ~98%). Per-model overrides (pi `compaction.modelOverrides`, `46bde88a1`; codex catalog `auto_compact_token_limit`). Scope option: count only tokens added after the carried prefix (codex `BodyAfterPrefix`).
- **Check points**: after user turn only (pi until Aug 2026) → **before every provider request incl. post-tool turns** ✔ pi (`56700d42e`) + pre-prompt + post-run. Rejected: proactive *abort mid-turn* when nearing threshold (pi removed `5a9d844f9`). codex: PreTurn, MidTurn "roll over" only when the model still needs to continue, opt-in PostTurn at idle (buffered, failure swallowed), and on **model switch** (compaction-compatibility hash changed or smaller window) using the *previous* model.
- **What's kept**: recent token budget verbatim (pi keepRecentTokens 20000, taken from Codex `compact.rs`) vs only summary vs pruning old tool outputs first (OpenCode-style "prune", considered in pi plan, not adopted) vs **only recent *user* messages verbatim (20k tokens) + summary; all assistant/tool items dropped** ✔ codex local; remote: retained user/hook/selected agent messages (64k) + opaque server item ✔ codex.
- **Summary placement**: summary as user message with preamble ✔ pi, ✔ codex (third-person "Another language model started to solve this problem…" handoff framing) / as system text / as assistant message / **opaque encrypted server item** ✔ codex remote.
- **Who summarizes**: client prompt to the session model ✔ pi, ✔ codex fallback vs **server-side compaction** (history + `CompactionTrigger` item → one encrypted `Compaction` output) ✔ codex preferred → [[compaction-locus]]. Third option: no summary, model-requested reset ✔ codex token-budget → [[model-requested-context-reset]].
- **Context re-injection**: canonical initial context (environment, permissions, world state) re-rendered and inserted before the last real user message ✔ codex.
- **Storage**: destructive rewrite vs append-only entry pointing at first kept entry ✔ pi (history never deleted; context rebuilt via [[context-projection]]) vs **checkpoint item carrying the full replacement history** in an append-only log, live history replaced in memory ✔ codex (`CompactedItem.replacement_history`).
- **Summarizer model**: session model ✔ pi (no cheap-summarizer setting; extensions can choose, e.g. Gemini Flash example), ✔ codex (but the *previous* model on model switch, with fallback to the current one) vs dedicated small model.
- **Reserve doubles as output budget**: pi reuses reserveTokens for summary `maxTokens` (0.8×) — couples trigger and summary size.
- **Async vs blocking**: blocking ✔ pi coding-agent; background soft-threshold ✔ pi durable → [[background-compaction]].
- **Manual**: `/compact [focus]` appends "Additional focus:" ✔ pi; `/compact` (no focus) routed by provider capability ✔ codex; whole prompt replaceable via config ✔ codex.
- **Plugin override**: pre-hook may cancel or supply its own summary ✔ pi (`session_before_compact`); Pre/Post compact command hooks may abort ✔ codex.
- **Compaction-survival contracts**: every feature storing state in history needs explicit retention across compaction (codex ~15 "Preserve X across compaction" commits) → [[compaction-drops-harness-state]].

## Implementations
- [[pi--auto-compaction|pi]] — fixed-headroom trigger (16384) checked before every request, post-run and pre-prompt; keep 20000 tokens; structured iterative summary as user message; append-only `compaction` entry with system-prompt snapshot; durable package re-implements as checkpointed task with background threshold.
- [[codex--auto-compaction|codex]] — 90%-of-window trigger at PreTurn/MidTurn/PostTurn/model-switch; remote server-side compaction for Responses providers, 9-line free-form handoff prompt otherwise; keeps ≤20k tokens of user messages verbatim; context re-injected after.

## Failures
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
- [[domain-biased-summarizer-prompt]]
- [[injected-summary-indistinguishable]]
- [[overflow-compaction-cascade]]
- [[auto-compact-threshold-exceeds-window]]
- [[compaction-drops-harness-state]]
- [[compaction-drops-pending-prompt]]
- [[compaction-loses-modality-or-structure]]
- [[summary-estimator-payload-mismatch]]
- [[estimator-undercounts-context]]
- Cross-group: [[compaction-pinned-to-unavailable-model]] (02) · [[deferred-tools-lost-on-resume]] (03) · [[steering-message-re-answered]] (01) · [[compaction-cancellation-races]], [[side-phase-input-lost]] (01-loop), `fork-boundary-loss` (08-state), `side-request-cache-pollution` (06-caching)
- [[proxied-stream-option-loss]] (01-loop) — Model calls routed through an indirection layer behaved like a different agent: the agent-core streamProxy…
- [[fork-boundary-loss]] (08-state) — Forking a path containing labels orphaned subtrees (entries parented on dropped label nodes) (#5669); forking…
- [[truncated-summary-persisted]] (05-context) — Summaries cut off at the output-token cap were saved as the session's compaction checkpoint; the missing half…
- [[oversized-trailing-tool-results-uncompactable]] (05-context) — When the trailing tool results alone exceeded the keep budget (e.g. several large reads after one assistant…
- [[overflow-judged-against-wrong-model]] (05-context) — After switching from a smaller-context model (e.g. Opus) to a larger one (e.g. Codex), the old model's…
- [[parallel-side-requests-single-slot-provider]] (05-context) — Split-turn compaction issued two overlapping summary generations (history + turn prefix); single-concurrency…

## Related
[[compaction-cut-point]] · [[split-turn-summary]] · [[structured-compaction-summary]] · [[iterative-summary-update]] · [[transcript-serialization-for-summary]] · [[summary-validation]] · [[file-op-tracking]] · [[background-compaction]] · [[overflow-recovery]] · [[token-estimation]] · [[context-projection]] · [[context-edit-overlay]] · [[cache-retention-control]] · [[session-handoff]] · [[durable-execution]] · [[model-requested-context-reset]] · [[world-state-diff-injection]] · [[compaction-locus]]
