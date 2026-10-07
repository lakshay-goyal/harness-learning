---
type: concept
stage: context
tier: candidate
aliases: [estimateContextTokens, estimateProjectedContextTokens, calculateContextTokens, estimateTokens, "chars/4", CHARS_PER_TOKEN, ESTIMATED_IMAGE_CHARS, hybrid-token-estimation, usage-anchored-context-estimate, get_total_token_usage, estimate_item_token_count, approx_token_count, APPROX_BYTES_PER_TOKEN, ServerReasoningIncluded, ContextWindowTokenStatus, effective_context_window_percent]
harnesses: [pi, codex]
---
Context size = the last trustworthy provider-reported usage + a character heuristic for everything appended after it; usage is invalidated at any boundary (compaction, context edit) that changed the prefix it measured.

## Why
- Tokenizers differ per provider and aren't available client-side; pure heuristics drift, pure provider usage is missing for the newest messages, for errored responses, and for providers that don't stream usage.
- Usage carried by messages kept across a compaction describes the *old, larger* prefix → compaction re-triggers immediately ([[stale-usage-drives-compaction]]).
- Requiring valid usage on the latest message starves compaction during error storms or with usage-less providers ([[compaction-starved-by-missing-usage]]).
- Any message kind the estimator ignores (custom messages, images) under-counts context ([[estimator-undercounts-context]]).

## Design space
- Provider usage only / heuristic only / **hybrid anchor + trailing estimate** ✔ pi, ✔ codex (last reported `total_tokens` + estimate of items after the last model-generated item).
- Heuristic ratio: chars/4 ("conservative") in coding-agent compaction vs **3.5 chars/token** in pi-ai request clamp (`27075fe07`) — two estimators coexist in pi and disagree.
- Unit: chars ✔ pi vs **model-visible bytes / 4, ceiling, excluding ids/metadata/JSON escaping** ✔ codex.
- Images: fixed 4800 chars (~1200 tok @4) per image ✔ pi; fixed 7,373 bytes (~1,844 tok) per resized image, or decoded 32-px patch count (≤10,000) for `detail: original`, cached ✔ codex.
- Hidden reasoning: add estimated encrypted reasoning tokens (`len*3/4 − 650`) unless the server says it already counted them (`ServerReasoningIncluded`) ✔ codex.
- Headroom: effective window = 95 % of context window ✔ codex; scope "only tokens after the carried prefix" (`BodyAfterPrefix`) so a large carried prefix doesn't re-trigger compaction ✔ codex.
- On overflow: pin usage to the full window so the next check fires ✔ codex.
- Invalidation: by position (usage entry before latest compaction/edit → pure estimate) ✔ pi coding-agent; by timestamp (assistant older than a later-inserted prefix message) ✔ pi-ai; durable: only usage appended after the head marker.
- Usage field: `totalTokens || input+output+cacheRead+cacheWrite` ✔ pi.
- Display: show unknown (`?/200k`) after compaction until next real usage ✔ pi.
- Exact local tokenizer (rejected/absent in pi; codex added tiktoken-rs `fd0673e457` and deleted it a month later `52d0ec4cd8`, [[no-local-tokenizer]]).

## Implementations
- [[pi--token-estimation|pi]] — coding-agent `estimateContextTokens`/`estimateProjectedContextTokens` (chars/4), pi-ai `estimate.ts` (3.5, timestamp guard), durable `estimateContext` anchored after head marker.
- [[codex--token-estimation|codex]] — reported usage + bytes/4 for newer items, prior-turn encrypted reasoning added, image patch estimates, 90 %/95 % thresholds.

## Failures
- [[stale-usage-drives-compaction]]
- [[compaction-starved-by-missing-usage]]
- [[estimator-undercounts-context]]
- [[summary-estimator-payload-mismatch]]
- [[auto-compact-threshold-exceeds-window]]
- [[image-content-poisoning]] (05-context) — One bad image block in history bricked the session: every later request was rejected (HTTP 400 / size errors)…

## Related
[[auto-compaction]] · [[max-tokens-context-clamp]] · [[usage-cost-accounting]] · [[context-projection]] · [[context-edit-overlay]] · [[compaction-cut-point]] · [[overflow-recovery]]
