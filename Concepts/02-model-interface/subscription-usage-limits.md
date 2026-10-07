---
type: concept
stage: model-interface
tier: variant
aliases: [RateLimitSnapshot, parse_all_rate_limits, x-codex-primary-used-percent, x-codex-active-limit, x-codex-credits-balance, codex.rate_limits, UsageLimitReached, QuotaExceeded, rate_limit_reached_type]
harnesses: [codex]
---
Track the plan and usage windows the provider reports (used percent, window length, reset time, credits) from response headers or stream events, show them to the user, and turn "limit reached" or "quota exhausted" into a terminal error that explains when access resets, instead of retrying.

## Why
- Subscription plans have rolling windows (primary and secondary) and credit balances. A retry loop cannot wait them out; it only burns time and hides the reset time ([[error-text-breaks-retry-classification]]).
- Throttling (`slow_down`, `rate_limit_exceeded`) and exhaustion (`insufficient_quota`, spend limits) can share HTTP 429. Only a structured code separates "retry soon" from "come back tomorrow".
- Users and long-running autonomous loops need advance warning (percent used) to pace work or stop, e.g. goal loops that block on a usage limit.

## Design space
- **Source**
  - None: only react to errors.
  - Response headers per request, all limit families parsed generically. ✔ codex
  - A stream event for transports without headers. ✔ codex: WebSocket `codex.rate_limits`.
  - A separate polling endpoint. ✔ codex: ChatGPT backend rate-limit resets.
- **Classification**
  - By status: 429 means retry.
  - By code, into throttled (retryable with delay), limit reached (terminal, with reset info), quota or spend exhausted (terminal) and plan doesn't include the feature (terminal). ✔ codex
- **On limit reached**
  - Generic error.
  - Typed error carrying plan, reset time, window and snapshot, stored as session state before failing. ✔ codex
  - Notify the workspace owner. Tried by codex and reverted the next day (`930e5adb7e` 2026-04-10).
- **Loop integration**
  - Usage limit as a blocking state for autonomous goal loops. ✔ codex (`0d344aca9b`)
- **pi:** absent (no subscription quota surface).

## Implementations
- [[codex--subscription-usage-limits|codex]] — `x-{limit}-{primary|secondary}-*` header families, credits headers, the `codex.rate_limits` WebSocket event, terminal `UsageLimitReached`/`QuotaExceeded`.

## Failures
- [[error-text-breaks-retry-classification]]

## Related
[[errors-as-stream-events]] · [[auto-retry-backoff]] · [[subscription-oauth-auth]] · [[usage-cost-accounting]] · [[persistent-goal-continuation]] · [[session-token-budget]]
