---
type: implementation
harness: codex
concept: usage-cost-accounting
commit: 622e9e3696
files: [codex-rs/codex-api/src/sse/responses.rs:128-156, codex-rs/protocol/src/protocol.rs:2299-2318, codex-rs/protocol/src/protocol.rs:2477-2522, codex-rs/app-server/src/turn_cost_worker.rs, codex-rs/backend-client/src/client/chatgpt_turn_cost.rs, codex-rs/otel/src/events/session_telemetry.rs:1139, codex-rs/core/src/rollout_budget.rs:62-66]
---
[[usage-cost-accounting]] in [[codex]]. Tokens are normalized client-side. **Cost is priced by the server.**

## Mechanism
- **Usage struct** `TokenUsage`, filled from `response.completed.usage` (`codex-rs/codex-api/src/sse/responses.rs:128-156`; `codex-rs/protocol/src/protocol.rs:2299-2318`):
  - `input_tokens` **includes** cached tokens;
  - `cached_input_tokens`, from `input_tokens_details.cached_tokens`;
  - `cache_write_input_tokens`, from `cache_write_tokens`;
  - `output_tokens`;
  - `reasoning_output_tokens`, a subset of output;
  - `total_tokens`;
  - `codex_rollout_budget_units`: provider-reported units for the shared rollout budget, never serialized out;
  - raw usage metadata, also preserved.
- **Derived values** (`codex-rs/protocol/src/protocol.rs:2477-2522`):
  - `non_cached_input = input - cached`.
  - `blended_total = non_cached_input + output`, the number shown to users.
  - Context-left percent subtracts `BASELINE_TOKENS` = 12_000 from both used and window, so the bar starts at 100% after the first prompt: "`BASELINE_TOKENS` should capture tokens that are always present in" the context.
- **No client price table** ([[no-client-price-table]]):
  - The app-server polls a turn-cost endpoint after turns end and emits `codex.turn_cost` with estimated USD (`codex-rs/app-server/src/turn_cost_worker.rs`: `POLL_INTERVAL` 150 s, `REQUEST_TIMEOUT` 15 s, `MAX_TRACKED_TURNS` 4096, `MAX_STALLED_POLL_ATTEMPTS` 5).
  - The ChatGPT variant returns `estimated_usage_usd_micros` per turn (`codex-rs/backend-client/src/client/chatgpt_turn_cost.rs`).
- **Cache telemetry only.** Cached input and cache-write tokens go to OTEL session telemetry (`codex-rs/otel/src/events/session_telemetry.rs:1139`). There is no per-turn miss detector ([[cache-miss-accounting]]).
- **Budget weighting.** [[session-token-budget]] charges output × sampling weight + non-cached input × prefill weight (`codex-rs/core/src/rollout_budget.rs:62-66`). It is a cost proxy with no dollar prices.

## Constants
| name | value | path:line |
|---|---|---|
| `BASELINE_TOKENS` (context-left baseline) | 12_000 | `codex-rs/protocol/src/protocol.rs:2477` |
| turn-cost `POLL_INTERVAL` | 150 s | `codex-rs/app-server/src/turn_cost_worker.rs` |
| turn-cost `REQUEST_TIMEOUT` / `MAX_TRACKED_TURNS` / `MAX_STALLED_POLL_ATTEMPTS` | 15 s / 4096 / 5 | `codex-rs/app-server/src/turn_cost_worker.rs` |

## Evolution
- `2edad72de3` 2026-07-16: `cache_write_input_tokens` added.
- `bb5054fe47` 2026-08-03: `codex_rollout_budget_units` captured from usage.
- `04caa22c82` 2026-08-17: turn cost (API key, OTLP only).
- `0cc80b8db5` 2026-08-20: turn cost for custom providers.

## Quirks
- The partition **overlaps**: input includes cached. pi's is disjoint. Summing `input + cached` would double-count.
- Cost is visible only after settlement, and only for providers that expose the turn-cost endpoint.

## Versus pi
- [[pi--usage-cost-accounting|pi]] prices every message locally from catalog rates: tiers, 1h cache writes, service tiers and fallback-model attribution.
- Codex delegates pricing to the vendor backend and keeps only token counts.

## Failures
- [[usage-double-counting]] (risk noted under Quirks; no codex incident found)
