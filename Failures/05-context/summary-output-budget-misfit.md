---
type: failure
concepts: [auto-compaction, branch-summary, thinking-level-abstraction]
harnesses: [pi]
---
**Symptom** — (a) Compaction requests failed with an API error because the requested summary `max_tokens` exceeded the model's maximum output; (b) branch summaries came back truncated because reasoning consumed the 2048-token cap.

**Root cause** — Summary budgets were derived only from harness settings (`0.8 × reserveTokens` = 13107; branch fixed 2048), not clamped to the model and not sized for reasoning models.

**Fix · [[pi]]**
- `3d9e14d74` 2026-05-11 (#4390) clamp summary output to `model.maxTokens` (`packages/coding-agent/src/core/compaction/compaction.ts:712-715`, `1090-1093`; durable `compaction.ts:139-142`).
- `e44d75c20` 2026-09-03 (#8845) branch summary cap `min(4096, model.maxTokens)` (`packages/coding-agent/src/core/compaction/branch-summarization.ts:345`).
- Related provider-side: `8df22faed`/`cd43b8a9c` `adjustMaxTokensForThinking` (Bedrock/Anthropic rejected `max_tokens ≤ budget_tokens`, compaction failed, #797) — see [[thinking-consumes-answer-budget]].

**Lesson** — Side-request output budgets must be clamped to the model's limits and leave room for reasoning.

Related: [[auto-compaction]] · [[branch-summary]] · [[truncated-summary-persisted]] · [[max-tokens-context-clamp]]
