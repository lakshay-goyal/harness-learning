---
type: implementation
harness: pi
concept: overflow-recovery
commit: b30a6dd77
files: [packages/coding-agent/src/core/agent-session.ts:2946, packages/coding-agent/src/core/agent-session.ts:3010, packages/coding-agent/src/core/agent-session.ts:1236, packages/coding-agent/src/core/agent-session.ts:1119, packages/coding-agent/src/core/agent-session.ts:3704, packages/ai/src/utils/overflow.ts:137, packages/ai/src/utils/overflow.ts:179, packages/durable/src/harness/generation.ts:459]
---
[[overflow-recovery]] in [[pi]].

## Mechanism
- Entry: post-run driver `_handlePostAgentRun` → (retryable error? → [[pi--auto-retry-backoff]]) → `_checkCompaction(msg, skipAborted=true, toolResults)` (`packages/coding-agent/src/core/agent-session.ts:1852-1890`, call `packages/coding-agent/src/core/agent-session.ts:1883`).
- **Retry/overflow split**: `_isRetryableError` returns false for any `isContextOverflow(message, contextWindow)` — "Context overflow is handled by compaction, not retry" (`packages/coding-agent/src/core/agent-session.ts:3704-3712`).
- **Detection** (`packages/coding-agent/src/core/agent-session.ts:2976-3008`), all gated on `sameModel` (`_modelForMessage` returns undefined if provider/model differ, `packages/coding-agent/src/core/agent-session.ts:615-619`; comment: switching opus→codex must not compact for the old model's overflow, `packages/coding-agent/src/core/agent-session.ts:2957-2961`):
  - `explicitOverflow = stopReason==="error" && isContextOverflow(msg)` — only if the assistant entry is still retained (no later compaction, no null `context_edit` on it).
  - silent overflow `isContextOverflow(msg, contextWindow)` only if `assistantUsageMatchesProjection` (assistant still projected, no later `context_edit`).
  - `recoverableLength = isRecoverableLength(msg, messageModel.maxTokens)` (`packages/ai/src/utils/overflow.ts:179-181`): `length` stop with `usage.output < desiredMaxOutput` (original, pre-clamp limit).
  - Classifier internals (25 regexes, `NON_OVERFLOW_PATTERNS` veto, Cerebras bodyless, z.ai silent `input+cacheRead > window` on `stop`, Xiaomi `length` + 0 output + ≥99% window, `packages/ai/src/utils/overflow.ts:37-171`) → [[context-overflow-detection]] (02).
- **Case 2 — completed response** (`willRetry = stopReason !== "stop"` false): `_runAutoCompaction("overflow", false)`, no retry because `agent.continue()` cannot continue from a completed assistant message (`packages/coding-agent/src/core/agent-session.ts:3010-3016`; `6b9f3f492`).
- **Case 1 — error/length**:
  - latch already set → emit `compaction_end{reason:"overflow", errorMessage}` + `session_compact_failed`: "Context overflow recovery failed after one compact-and-retry attempt. Try reducing context or switching to a larger-context model." / "Truncated response recovery failed after one compact-and-retry attempt." (`packages/coding-agent/src/core/agent-session.ts:3018-3038`; wording `c7c763f5c`).
  - else set `_overflowRecoveryAttempted = true`; `_omitRecoveryAttempt(msg, toolResults)` appends `context_edit{targetId, replacement:null}` for the failed assistant + its tool results (raw log keeps them; throws if a projected target has no source entry) (`packages/coding-agent/src/core/agent-session.ts:1236-1252`); `_runAutoCompaction("overflow", true)`; on success set `_failedResponse = msg` (virtual-router `reason:"retry"`, `packages/coding-agent/src/core/agent-session.ts:819-828`) → driver calls `agent.continue()` (`packages/coding-agent/src/core/agent-session.ts:3040-3045`).
- **Latch reset**: any user `message_start` (`packages/coding-agent/src/core/agent-session.ts:1119`) and any assistant `message_end` with stopReason ∉ {error, length} (`packages/coding-agent/src/core/agent-session.ts:1169-1171`). Field declared `packages/coding-agent/src/core/agent-session.ts:405`.
- **Cut interplay**: the omitted attempt forms a context-invisible suffix; `findProjectedCutPoint` advances the cut past it so an over-budget recovered input can still be summarized while the omission edits stay in force ([[pi--compaction-cut-point]]; doc `packages/coding-agent/docs/compaction.md:138`).
- **Docs contract** for custom providers: normalize unknown overflow messages to `context_length_exceeded`; never rewrite rate limits as overflow (`packages/coding-agent/docs/custom-provider.md:152-154`).

## Constants
| name | value | path:line |
|---|---|---|
| recovery attempts per user turn | 1 (`_overflowRecoveryAttempted`) | `packages/coding-agent/src/core/agent-session.ts:405`, `packages/coding-agent/src/core/agent-session.ts:3018` |
| length-stop overflow threshold | `input+cacheRead ≥ 0.99·window`, output 0 | `packages/ai/src/utils/overflow.ts:160-168` |
| recoverable length | `output < desiredMaxOutput` | `packages/ai/src/utils/overflow.ts:179-181` |
| durable overflow compactions per generation | 1 (`compacted` checkpoint) | `packages/durable/src/harness/generation.ts:464-471` |

## Evolution
- 2025-12-09 `a38e61909` overflow recovery introduced; `5a9d844f9` simplified to "overflow (auto-retry) / threshold (no retry)" using new `Agent.continue()`.
- 2026-01-07 `615ed0ae2` (#535) skip overflow from a different model.
- 2026-01-29 `25707f9ad` (#1038) 429 no longer overflow; 2026-03-30 `a3bf1eb39` (#2699) Bedrock throttling veto; 2026-09-18 `661619e87` (#9482) bodyless 400/413 Cerebras-only (classifier side, 02).
- 2026-03-03 `6b4b92042` (#1319) "stop overflow auto-compaction cascades" — one-shot latch.
- 2026-06-18 `6b9f3f492` (#5720) successful overflowing response: compact, don't retry.
- 2026-06-25 `f14b3594c` (#4290) visible length-stop errors; 2026-08-03 `32850ef7c` (#7540) resume after context-limited length stops (first tried "prompt usage within 1% of window", replaced by `isRecoverableLength`; Responses `incomplete` normalized so only `max_output_tokens` is a length stop); 2026-08-17 `c7c763f5c` (#8130) clarified failure message.
- 2026-09-21 `466db0fec` failed attempts omitted via `context_edit` instead of lingering in context.
- 2026-09-30 `ed0d6b91b` durable: overflow compacts once, "An overflow is never retried: only a compaction can make the next request fit."

## Evidence commits
`a38e61909` `5a9d844f9` `615ed0ae2` `25707f9ad` `a3bf1eb39` `661619e87` `6b4b92042` `6b9f3f492` `f14b3594c` `32850ef7c` `c7c763f5c` `466db0fec` `ed0d6b91b` `39b1bf7b6`

## Quirks
- Silent-overflow check uses `input + cacheRead`, not `cacheWrite` (`packages/ai/src/utils/overflow.ts:154`) although `32850ef7c`'s message mentions cache-write tokens — divergence (unverified intent).
- Latch is per user message, so a long autonomous tool loop gets only one recovery for the whole run.
- Ollama silent truncation is undetectable (`packages/ai/src/utils/overflow.ts:118-120`).
- Anthropic 413 `request_too_large` (byte size, e.g. images) counts as overflow (`39b1bf7b6`, #2734) — compaction can't shrink an oversized *image*, so recovery may fail once and stop (inferred).

## Durable variant (packages/durable)
- Classification in generation (`packages/durable/src/harness/generation.ts:452-482`): `stop`/`length`/`toolUse` without calls → answer (no recoverable-length path); `error` + `isContextOverflow` + compaction enabled + `compacted === undefined` + `selectCut` finds a cut → append the overflowed assistant, create owned `pi.compaction{reason:"overflow"}`, wait (`allSettled`), re-`prepare` with `overflow` text. Overflow is excluded from the retry branch. No cut → falls to error handling.

## Failures
[[overflow-compaction-cascade]] · [[completed-response-retried-after-overflow]] · [[overflow-judged-against-wrong-model]] · [[length-stop-recovery]] · cross-group: [[rate-limit-misread-as-overflow]], [[overflow-message-not-recognized]]
