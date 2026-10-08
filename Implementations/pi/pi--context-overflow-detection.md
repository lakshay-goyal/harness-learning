---
type: implementation
harness: pi
concept: context-overflow-detection
commit: b30a6dd77
files: [packages/ai/src/utils/overflow.ts:37, packages/ai/src/utils/overflow.ts:65, packages/ai/src/utils/overflow.ts:67, packages/ai/src/utils/overflow.ts:118, packages/ai/src/utils/overflow.ts:137, packages/ai/src/utils/overflow.ts:173, packages/coding-agent/src/core/agent-session.ts:2950, packages/coding-agent/src/core/agent-session.ts:3704, packages/durable/src/harness/generation.ts:460]
---
[[context-overflow-detection]] in [[pi]].

## Mechanism
`isContextOverflow(message, contextWindow?)` (packages/ai/src/utils/overflow.ts:137-171) — three cases:
1. `stopReason==="error"` + `errorMessage` matches `OVERFLOW_PATTERNS` and no `NON_OVERFLOW_PATTERNS` veto.
2. **Silent overflow** (z.ai accepts oversized input): `stopReason==="stop"` and `input + cacheRead > contextWindow` (:152-158) — cacheWrite NOT included.
3. **Length-stop overflow** (Xiaomi MiMo truncates input then returns `length` with 0 output): `stopReason==="length" && output===0 && input+cacheRead >= 0.99·contextWindow` (:160-168; a44622670).

**OVERFLOW_PATTERNS** (overflow.ts:37-63; 25 regexes, + 1 Cerebras-only bodyless pattern at :65):

| pattern | provider |
|---|---|
| `/prompt (?:is )?too long/i` | Anthropic, z.ai |
| `/prompt exceeds max length/i` | z.ai CN |
| `/request_too_large/i` | Anthropic 413 byte-size (images) |
| `/input is too long for requested model/i` | Bedrock |
| `/exceeds the context window/i` | OpenAI Completions & Responses |
| `/exceeds (?:the )?(?:model'?s )?maximum context length(?: of [\d,]+ tokens?\|\s*\([\d,]+\))/i` | LiteLLM / OpenAI-compatible |
| `/input token count.*exceeds the maximum/i` | Gemini |
| `/maximum prompt length is \d+/i` | xAI |
| `/reduce the length of the messages/i` | Groq |
| `/maximum context length is \d+ tokens/i` | OpenRouter |
| `/exceeds (?:the )?maximum allowed input length of [\d,]+ tokens?/i` | OpenRouter/Poolside |
| `/input \(\d+ tokens\) is longer than the model'?s context length \(\d+ tokens\)/i` | Together |
| `/exceeds the limit of \d+/i` | GitHub Copilot |
| `/exceeds the available context size/i` | llama.cpp |
| `/greater than the context length/i` | LM Studio |
| `/context window exceeds limit/i` | MiniMax |
| `/exceeded model token limit/i` | Kimi For Coding |
| `/too large for model with \d+ maximum context length/i` | Mistral |
| `/prompt has [\d,]+ tokens?, but the configured context size is [\d,]+ tokens?/i` | DS4 |
| `/model_context_window_exceeded/i` | z.ai finish_reason as error |
| `/prompt too long; exceeded (?:max )?context length/i` | Ollama |
| `/range of input length should be/i` | DashScope/Qwen |
| `/context[_ ]length[_ ]exceeded/i`, `/too many tokens/i`, `/token limit exceeded/i` | generic fallbacks |

- **Cerebras bodyless** `/^4(?:00|13)\s*(?:status code)?\s*\(no body\)/i` only when `provider === "cerebras"` (:65, 146-148; 661619e87).
- **NON_OVERFLOW_PATTERNS** veto evaluated first: `/^(Throttling error|Service unavailable):/i` (Bedrock formatted prefixes from `formatBedrockError`), `/rate limit/i`, `/too many requests/i` — Bedrock "ThrottlingException: Too many tokens" otherwise matches `/too many tokens/` (:67-80, 141; a3bf1eb39) → [[rate-limit-misread-as-overflow]].
- Ollama silent truncation explicitly undetectable (:118-120). Custom-provider recipe "add a regex or check yourself" (:122-131); docs: normalize unknown overflow messages to `context_length_exceeded`, never rewrite rate limits as overflow (packages/coding-agent/docs/custom-provider.md:152-154).
- `isRecoverableLength(msg, desiredMaxOutput)`: `length` stop with output < intended max → caller may do **one** compact-and-retry (:173-181; 32850ef7c #7540 — its commit message describes a "prompt usage within 1% of window" heuristic, but the merged code is `output < desiredMaxOutput`; heuristic superseded within the PR (inferred)). Responses `incomplete` reasons normalized so only `max_output_tokens` is a length stop (openai-responses-shared.ts:779-809) → [[length-stop-recovery]].
- Bedrock maps `model_context_window_exceeded` stop → `length` (bedrock-converse-stream.ts:1177-1192).
- **Consumers**: coding-agent `_checkCompaction` (packages/coding-agent/src/core/agent-session.ts:2950-3046): skip if message from a different model (user switched to larger model, :2957-2964; 615ed0ae2 #535); skip if older than latest compaction (:2966-2974); explicit = error overflow; silent only if assistant still projected + no later context edit (`assistantUsageMatchesProjection`); `recoverableLength = isRecoverableLength(msg, model.maxTokens)` (:3002-3008); successful-but-overflowing → compact, no retry (:3012-3016; 6b9f3f492); one-shot `_overflowRecoveryAttempted` (:3018-3038; 6b4b92042 #1319) → [[overflow-recovery]].
- Retry path excludes overflow before retry classification: "Context overflow is handled by compaction, not retry" (agent-session.ts:3704-3712); pi-ai contract: caller must handle overflow **before** retry (packages/ai/src/utils/retry.ts:243-251) → [[auto-retry-backoff]].

## Durable variant (packages/durable)
- `pi.generation` classification: context-overflow `error` compacts **once** then re-prepares; "An overflow is never retried: only a compaction can make the next request fit" (packages/durable/src/harness/generation.ts:459-478, 460, 464-471).

## Constants
| name | value | path:line |
|---|---|---|
| OVERFLOW_PATTERNS | 25 regexes (+1 Cerebras-only, :65) | packages/ai/src/utils/overflow.ts:37-63 |
| length-stop overflow threshold | 0.99 × contextWindow | packages/ai/src/utils/overflow.ts:163-168 |
| overflow recovery attempts | 1 per user turn | packages/coding-agent/src/core/agent-session.ts:3018-3038 |

## Evolution
- 2025-12-06 `a325c1c7d` `isContextOverflow` regex table + silent overflow.
- 2026-01-07 `615ed0ae2` same-model guard (#535); 2026-01-08 `946efe4b4` `context_length_exceeded`.
- 2026-01-29 `25707f9ad` 429 removed from bodyless overflow regex (#1038).
- 2026-03-03 `6b4b92042` stop overflow compaction cascades (#1319).
- 2026-03-08 `ade6a35e7` z.ai (#1937); 2026-03-27 `bc8eb74b8` Ollama (#2626); 2026-03-30 `a3bf1eb39` Bedrock throttling veto (#2699); 2026-04-01 `39b1bf7b6` `request_too_large` (#2734).
- 2026-05-02 `a44622670` MiMo length-stop heuristic; 2026-05-16 `7c5c3d6fd` LiteLLM (#4563); 2026-05-26 `fa1180b6b` Poolside (#4943); 2026-06-13 `121f0edbf` parenthesized (#5677); 2026-06-18 `6b9f3f492` don't retry successful overflow (#5720); 2026-07-02 `21cb3807e` DS4 (#6262).
- 2026-08-03 `32850ef7c` `isRecoverableLength` (#7540).
- 2026-09-18 `661619e87` Cerebras-only bodyless (#9482); 2026-09-20 `0e283203c` z.ai prompt-too-long (#9805); 2026-09-30 `3dd803d7e` Z.AI CN (#10208).

## Evidence commits
a325c1c7d · 615ed0ae2 · 946efe4b4 · 25707f9ad · 6b4b92042 · ade6a35e7 · bc8eb74b8 · a3bf1eb39 · 39b1bf7b6 · a44622670 · 7c5c3d6fd · fa1180b6b · 121f0edbf · 6b9f3f492 · 21cb3807e · 32850ef7c · 661619e87 · 0e283203c · 3dd803d7e

## Quirks
- Silent-overflow sums `input + cacheRead` but not `cacheWrite` (overflow.ts:154) while `32850ef7c`'s message mentions cache-write — intentional divergence unverified.
- Regex-over-text classification is open-ended whack-a-mole (~10 pattern additions); structured error kinds would avoid it.
- Generic `/too many tokens/i` remains and depends on the veto list being exhaustive.

## Failures
- [[rate-limit-misread-as-overflow]] · [[overflow-message-not-recognized]] · [[silent-overflow-undetected]] · [[length-stop-recovery]]
