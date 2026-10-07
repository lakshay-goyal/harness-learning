---
type: concept
stage: failure-handling
tier: candidate
aliases: [isContextOverflow, OVERFLOW_PATTERNS, NON_OVERFLOW_PATTERNS, isRecoverableLength, overflow-error-classification, overflow-false-positive-veto, recoverable-length-stop, silent overflow, ContextOverflowError, parseAPICallError, isContextOverflowFailure]
harnesses: [pi, opencode]
---
Decide that a provider response means "the context no longer fits". The signals are:
- error text matched against a catalogue of provider phrasings, with exclusions that are evaluated first;
- silent overflow, where reported usage exceeds the window on a successful stop;
- truncation-style length stops, where the output came back empty or ended before the intended cap.

## Why
- Overflow needs compaction. Retrying the same request is pointless, and a misclassification in either direction is expensive:
  - Rate limits or throttling read as overflow trigger needless lossy compaction ([[rate-limit-misread-as-overflow]]).
  - Unrecognized overflow phrasings leave the agent retrying or stuck ([[overflow-message-not-recognized]]).
- Some providers never send an overflow error at all. They accept oversized input silently, or truncate the input and return a length stop with zero output ([[silent-overflow-undetected]]).
- Request byte limits, e.g. from images, are a form of overflow too.

## Design space
- **Signal source**
  - A structured error code from the provider. This is rarely available.
  - A regex catalogue over the error message. *pi chose this:* 25 patterns (`packages/ai/src/utils/overflow.ts:38-62`), plus a provider-gated heuristic for Cerebras bodyless 400/413.
  - Usage arithmetic: `input + cacheRead > window`.
  - A length stop at or below the intended maximum.
- **False positives**
  - None.
  - An exclusion list checked before overflow patterns: throttling and rate-limit text. *pi chose this:* a3bf1eb39.
  - Removing 429 from the bodyless heuristic: 25707f9ad.
  - Scoping provider-specific heuristics to that provider: 661619e87.
- **Length stops**
  - Treat every length stop as overflow.
  - "Prompt within 1% of the window". *pi tried this,* then replaced it in 32850ef7c.
  - A length stop below the intended output cap gets exactly one compact-and-retry. *pi chose this.*
- **Undetectable cases**
  - Ollama silent truncation is documented as unobservable.
  - Custom providers are told to normalize their overflow messages.
- **Pre-request usage check**: compare the last finished step's tokens with `limit.input − reserve` before sending (opencode).
- **Summarizer reserve**: `min(20k, maxOutput)` kept free so the compaction request itself fits (opencode `0fd6f365be`).

## Implementations
- [[pi--context-overflow-detection|pi]] — `packages/ai/src/utils/overflow.ts` provides `OVERFLOW_PATTERNS`, `NON_OVERFLOW_PATTERNS`, the silent and length-stop cases, and `isRecoverableLength`. The coding-agent and durable generation call these, with same-model and post-compaction guards.
- [[opencode--context-overflow-detection|opencode]] — usage, pre-request and error paths; shared `isContextOverflow` in `packages/llm` (27 regexes + rate-limit exclusions).

## Failures
- [[rate-limit-misread-as-overflow]]
- [[overflow-message-not-recognized]]
- [[silent-overflow-undetected]]
- [[length-stop-recovery]]

## Related
[[overflow-recovery]] · [[auto-compaction]] · [[token-estimation]] · [[errors-as-stream-events]] · [[auto-retry-backoff]] · [[max-tokens-context-clamp]]
