---
type: failure
concepts: [thinking-level-abstraction, max-tokens-context-clamp]
harnesses: [pi]
---
**Symptom** On OpenAI-compatible servers (vLLM, Qwen/SGLang, llama.cpp), "reasoning and the answer share max_tokens … a reasoning-heavy turn can consume the whole response and emit no answer" (`packages/ai/src/types.ts:104-105`). The same risk applies to budget providers when the thinking budget ≥ maxTokens.

**Root cause** The thinking budget was either uncapped (server default) or not bounded by the output ceiling.

**Fix · [[pi]]**
- `d07889da0` 2026-08-05: `thinking_token_budget` on openai-completions (#7638).
- `b23741269` 2026-08-18: generalized `thinkingTokenBudgetField` (vLLM `thinking_token_budget`, Qwen/SGLang `thinking_budget`, llama.cpp `thinking_budget_tokens`; `types.ts:854-861`) and named `MIN_ANSWER_TOKENS = 1024` (`packages/ai/src/api/simple-options.ts:67-68`). Budget is clamped so ≥1024 tokens remain under `max_tokens ?? max_completion_tokens ?? model.maxTokens` (`packages/ai/src/api/openai-completions.ts:1020-1032`) (#8275).
- Shared `adjustMaxTokensForThinking` shrinks the budget when maxTokens ≤ budget (`simple-options.ts:92-108`). Anthropic uses `min(budget, max(0, maxTokens − 1024))` (`anthropic-messages.ts:961-975`).

**Lesson** When thinking and the answer share one output ceiling, always reserve answer tokens.

Related: [[thinking-level-abstraction]] · [[max-tokens-context-clamp]] · [[output-token-cap-misbudgeted]] · [[pi--thinking-level-abstraction|pi]]
