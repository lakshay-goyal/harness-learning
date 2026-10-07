---
type: concept
stage: context
tier: candidate
aliases: [estimateContextTokens, estimateProjectedContextTokens, calculateContextTokens, estimateTokens, "chars/4", CHARS_PER_TOKEN, ESTIMATED_IMAGE_CHARS, hybrid-token-estimation, usage-anchored-context-estimate]
harnesses: [pi]
---
Context size = the last trustworthy provider-reported usage + a character heuristic for everything appended after it; usage is invalidated at any boundary (compaction, context edit) that changed the prefix it measured.

## Why
- Tokenizers differ per provider and aren't available client-side; pure heuristics drift, pure provider usage is missing for the newest messages, for errored responses, and for providers that don't stream usage.
- Usage carried by messages kept across a compaction describes the *old, larger* prefix → compaction re-triggers immediately ([[stale-usage-drives-compaction]]).
- Requiring valid usage on the latest message starves compaction during error storms or with usage-less providers ([[compaction-starved-by-missing-usage]]).
- Any message kind the estimator ignores (custom messages, images) under-counts context ([[estimator-undercounts-context]]).

## Design space
- Provider usage only / heuristic only / **hybrid anchor + trailing estimate** ✔ pi.
- Heuristic ratio: chars/4 ("conservative") in coding-agent compaction vs **3.5 chars/token** in pi-ai request clamp (`27075fe07`) — two estimators coexist in pi and disagree.
- Images: fixed 4800 chars (~1200 tok @4) per image ✔ pi.
- Invalidation: by position (usage entry before latest compaction/edit → pure estimate) ✔ pi coding-agent; by timestamp (assistant older than a later-inserted prefix message) ✔ pi-ai; durable: only usage appended after the head marker.
- Usage field: `totalTokens || input+output+cacheRead+cacheWrite` ✔ pi.
- Display: show unknown (`?/200k`) after compaction until next real usage ✔ pi.
- Exact local tokenizer (rejected/absent in pi).

## Implementations
- [[pi--token-estimation|pi]] — coding-agent `estimateContextTokens`/`estimateProjectedContextTokens` (chars/4), pi-ai `estimate.ts` (3.5, timestamp guard), durable `estimateContext` anchored after head marker.

## Failures
- [[stale-usage-drives-compaction]]
- [[compaction-starved-by-missing-usage]]
- [[estimator-undercounts-context]]

## Related
[[auto-compaction]] · [[max-tokens-context-clamp]] · [[usage-cost-accounting]] · [[context-projection]] · [[context-edit-overlay]] · [[compaction-cut-point]] · [[overflow-recovery]]
