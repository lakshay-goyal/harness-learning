---
type: implementation
harness: opencode
concept: context-overflow-detection
commit: ecc4916b5a
files: [packages/opencode/src/session/overflow.ts:8-34, packages/opencode/src/session/processor.ts:491-496, packages/opencode/src/session/prompt.ts:1161-1167, packages/opencode/src/provider/error.ts:102-193, packages/llm/src/provider-error.ts:4-43, packages/llm/src/route/executor.ts:253-265, packages/core/src/session/runner/llm.ts:286-297]
---
[[context-overflow-detection]] in [[opencode]].

## Mechanism

### Legacy runtime — three paths
- **Usage path**: at every `step-finish` (non-summary), `tokens.total || input+output+cache.read+cache.write ≥ usable` → `needsCompaction`; `Stream.takeUntil` stops, process returns `compact` (`packages/opencode/src/session/processor.ts:491-496`).
- **Pre-request path**: before calling the model, `compaction.isOverflow(lastFinished.tokens)` → `compaction.create({auto:true})`, continue (`packages/opencode/src/session/prompt.ts:1161-1167`).
- **Error path**: `parseAPICallError` → overflow if `isContextOverflow(message)` OR HTTP 413 OR body `error.code === "context_length_exceeded"`; streamed `context_length_exceeded` likewise (`packages/opencode/src/provider/error.ts:102-193`). `halt` turns it into `needsCompaction`, or a terminal error when `compaction.auto === false`.
- `usable` = `limit.input − reserved` when the catalog has a separate input limit, else `context − maxOutputTokens`; `reserved` = `compaction.reserved ?? min(20 000, maxOutputTokens)`; `limit.context === 0` disables overflow (`packages/opencode/src/session/overflow.ts:8-34`).

### v2 runtime / shared classifier (`packages/llm/src/provider-error.ts`)
- 27 provider regexes + `^4(00|13) (no body)` heuristic, vetoed by exclusions `^(throttling error|service unavailable):`, `rate limit`, `too many requests` (`packages/llm/src/provider-error.ts:4-38`). Legacy imports the same `isContextOverflow`.
- Applied at HTTP 4xx bodies (`packages/llm/src/route/executor.ts:253-265`) and Anthropic / Responses / Bedrock in-stream errors.
- Runner: an overflow provider error before any assistant output triggers `compactAfterOverflow` once; a second overflow after that compaction is a defect (`packages/core/src/session/runner/llm.ts:286-297,370`).

## Constants
| name | value | path:line |
|---|---|---|
| `COMPACTION_BUFFER` (reserve) | 20 000 | `packages/opencode/src/session/overflow.ts:8` |
| unknown window | `limit.context ?? 0` → no overflow check | `packages/opencode/src/session/overflow.ts:12,29` |

## Evolution
- 2026-01-15 `8d720f9463` input limit used; 2026-02-10 `0fd6f365be` reserve buffer so the summarizer still fits.
- 2026-02-10 `d98bd4bd52` generic `too many tokens` / `token limit exceeded` removed as overcorrecting (rate limits matched).
- 2026-03-02 `be20f865ac` 413 recovered via auto-compaction; 2026-03-16 `e718db624f` `code: context_length_exceeded`; 2026-03-18 `56102ff642` vLLM.
- 2026-06-04 `7e09660c3b` overflow respected `compaction.auto: false`; 2026-06-05 `820c984d47` classifier moved into `packages/llm`.
- 2026-07-07 `adf178a6b9` z.ai; 2026-07-19 `2a097f3af7` +7 patterns and the exclusion list (generic patterns back, guarded).

## Quirks / drift
- Usage check uses the previous step's tokens; tool results added after it are uncounted, so the next request can still overflow and must fall back to the error path (inference).

Failures: [[overflow-message-not-recognized]] · [[rate-limit-misread-as-overflow]].

Contrast: [[pi--context-overflow-detection|pi]] uses the same regex + exclusion-list shape; opencode adds a pre-request usage check and an input-limit-aware reserve.
