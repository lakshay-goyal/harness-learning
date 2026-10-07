---
type: implementation
harness: codex
concept: subscription-usage-limits
commit: 622e9e3696
files: [codex-rs/codex-api/src/rate_limits.rs:23-101, codex-rs/codex-api/src/rate_limits.rs:135-170, codex-rs/codex-api/src/rate_limits.rs:181-193, codex-rs/codex-api/src/rate_limits.rs:219-231, codex-rs/codex-api/src/sse/responses.rs:43, codex-rs/codex-api/src/sse/responses.rs:78, codex-rs/core/src/session/turn.rs:1692-1698, codex-rs/codex-api/src/sse/responses_error.rs:47-99, codex-rs/backend-client/src/client/rate_limit_resets.rs]
---
[[subscription-usage-limits]] in [[codex]].

## Mechanism
- **Header families** `x-{limit_id}-{primary|secondary}-{used-percent|window-minutes|reset-at}`, plus `-limit-name`. The default limit id is `codex`. Every family found in the headers is parsed into a per-limit snapshot (`parse_all_rate_limits`, `codex-rs/codex-api/src/rate_limits.rs:23-101`). The active limit is named by `x-codex-active-limit`.
- **Credits** come from `x-codex-credits-has-credits`, `x-codex-credits-unlimited` and `x-codex-credits-balance` (`codex-rs/codex-api/src/rate_limits.rs:219-231`).
- **Promo and reached type** come from `x-codex-promo-message` and `x-codex-rate-limit-reached-type` (`codex-rs/codex-api/src/rate_limits.rs:181-193`).
- **Delivery:**
  - Over SSE, snapshots are emitted as `ResponseEvent`s *before* the response events (`codex-rs/codex-api/src/sse/responses.rs:43,78`).
  - Over WebSockets, a `codex.rate_limits` event carries the same data (`codex-rs/codex-api/src/rate_limits.rs:135-170`).
- **Limit reached → terminal, explained error.** An HTTP 429 with `usage_limit_reached` becomes `UsageLimitReached`, carrying `plan_type`, `resets_at`, `limit_window_minutes`, a rate-limit snapshot and the promo message. The turn stores the snapshot (`sess.update_rate_limits`) before failing (`codex-rs/core/src/session/turn.rs:1692-1698`). It is never retried (`retry_delay` → `None`).
- **Quota and spend codes are terminal too**: `insufficient_quota`, `credit_balance_exhausted`, `organization_spend_limit_exceeded`, `project_spend_limit_exceeded` → QuotaExceeded; `usage_not_included` → UsageNotIncluded (`codex-rs/codex-api/src/sse/responses_error.rs:47-99`).
- **Throttling stays retryable.** `rate_limit_exceeded` and `slow_down` (since `31ffe2bc9a`) are RateLimitExceeded with a server delay.
- **Resets endpoint.** The ChatGPT backend also exposes rate-limit resets (`codex-rs/backend-client/src/client/rate_limit_resets.rs`).

## Evolution
- `0c647bc566` 2025-11-06: `insufficient_quota` no longer retried.
- `df000da917` 2026-02-04: `codex.rate_limits` WebSocket event.
- `fdd0cd1de9` 2026-02-10 (#11260): multiple named limits.
- `930e5adb7e` 2026-04-10: "notify workspace owner" on a usage limit, reverted the next day.
- `139fa8b8f2` 2026-04-17: rate-limit reached type.
- `0d344aca9b` 2026-05-18: usage limit as a goal-blocking state ([[persistent-goal-continuation]]).
- `4df8027a97` 2026-07-14: workspace spend controls.
- `102fc57e4a` 2026-09-10: split HTTP quota from rate limit.
- `31ffe2bc9a` 2026-09-15: `slow_down` retryable; spend and credit codes terminal.

## Versus pi
- pi has no subscription quota surface; usage-limit errors go through its generic retry classifier.

## Failures
- [[error-text-breaks-retry-classification]]
