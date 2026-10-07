---
type: concept
stage: compaction
tier: candidate
aliases: [model-initiated-context-reset, context-budget-reminder, Feature::TokenBudget, token_budget, new_context, get_context_remaining, start_new_context_window, TokenBudgetReminder, AutoCompactFallbackPrompt, history-notes, compact_token_budget]
harnesses: [codex]
---
The model sees how much context remains (reminder + tool) and can itself ask the harness to start a fresh context window; the "compaction" is a reset with no summary, and continuity is carried by model-maintained notes and a searchable archive of prior windows.

## Why
- A summary costs tokens and is lossy in ways the model can't steer; the model often knows best when a window is no longer useful ("without spending tokens on a compaction summary", `87ab01834a`).
- A model that is never told its remaining budget gets cut off mid-task by a forced compaction; a reminder + last-chance prompt lets it save state first.
- A reset without memory would lose everything → needs durable model-owned notes and retrievable history.

## Design space
- Harness-only trigger, LLM summary ✔ pi ([[auto-compaction]]), codex default local/remote compaction.
- **Remaining-token reminder injected once under a threshold** ✔ codex (`TokenBudgetReminder`, template with `{n_remaining}`); **last-chance fallback prompt at zero before forced reset** ✔ codex (`AutoCompactFallbackPrompt`, limit raised by a fallback buffer only when that prompt exists).
- **Model-callable tools**: `get_context_remaining` → `{tokens_left}`; `new_context` (reset, "Does not clear, reset, or otherwise affect environment state.") ✔ codex.
- Reset content: fresh initial context only (+ retained client developer messages within 64k), no summary, no user messages ✔ codex vs summary + kept tail (classic compaction).
- Continuity store: backend-hosted `history.*` (list/read/search prior windows) + `notes.*` (model-written files) tools, "private model-only state" ✔ codex (`history-notes` extension).
- Policy ownership: thresholds/templates from model catalog `model_messages.token_budget` ✔ codex → [[per-model-system-prompt]].
- Interaction with other compaction: token-budget sessions skip post-turn summarizing compaction and guardian overflow compaction ✔ codex.

## Implementations
- [[codex--model-requested-context-reset|codex]] — `Feature::TokenBudget` (under development; ChatGPT Plus/Pro/ProLite/ProMax only): reminder + fallback prompt, `new_context` / `get_context_remaining` tools, `compact_token_budget` reset lifecycle, `history-notes` backend tools.

## Failures
- (none recorded)

## Related
[[auto-compaction]] · [[token-estimation]] · [[session-token-budget]] · [[world-state-diff-injection]] · [[cross-session-memory]] · [[overflow-recovery]] · [[compaction-locus]]
