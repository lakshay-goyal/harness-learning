---
type: implementation
harness: codex
concept: install-telemetry
commit: 622e9e3696
files: [codex-rs/config/src/types.rs:622, codex-rs/config/src/types.rs:670, codex-rs/otel/src/config.rs:9, codex-rs/otel/src/agent_response.rs:18, codex-rs/analytics/src/client.rs:88, codex-rs/config/src/config_toml.rs:530, codex-rs/feedback/src/lib.rs:53, codex-rs/install-context/src/lib.rs:55, codex-rs/terminal-detection/src/lib.rs:1]
---
[[install-telemetry]] in [[codex]].

## Mechanism
- **OTEL defaults**: log exporter None, trace exporter None, **metrics exporter Statsig** (`codex-rs/config/src/types.rs:670-682`); Statsig resolves to OTLP/HTTP JSON at `https://ab.chatgpt.com/otlp/v1/metrics` with an embedded client key, disabled in debug builds (`codex-rs/otel/src/config.rs:9-32`) → metrics are opt-out.
- **Opt-in sensitive logs**: `log_user_prompt`, `log_agent_responses` (main + spawned sub-agent responses, capped 64 KiB), `log_guardian_assessments` (64 KiB) (`codex-rs/config/src/types.rs:622-633`; `codex-rs/otel/src/agent_response.rs:18`; `codex-rs/otel/src/guardian_assessment.rs:13`).
- **`[otel]` keys** (`OtelConfigToml`, `codex-rs/config/src/types.rs:622-646`): `tool_result` (byte limit for tool-result log output, independent of model-visible output), `log_user_prompt`, `log_agent_responses`, `log_guardian_assessments`, `environment` (default `dev`), `exporter` / `trace_exporter` / `metrics_exporter` (`OtelExporterKind`), `span_attributes` (added to every exported span), `tracestate` (W3C tracestate upserts).
- **Metric names** seen: `codex.multi_agent.spawn{role,version}`, `codex.multi_agent.result_delivery{outcome}`, `codex.multi_agent.wait.duration_ms{outcome}`, `codex.multi_agent.nickname_pool_reset`, spawn-failure reasons limit_reached / invalid_request / thread_not_found / unsupported_operation / internal (`codex-rs/core/src/tools/handlers/multi_agents_common.rs` `record_collab_spawn_failure`); goal metrics `codex.goal.budget_limited`, `codex.goal.usage_limited` (`codex-rs/otel/src/metrics/names.rs:52-53`). Inter-agent message tracing target `codex_otel.agent_communication` logs only encrypted content or "[plaintext]" (`codex-rs/core/src/agent_communication.rs:9`, `:45-70`). Cache read/write tokens reported (`codex-rs/otel/src/events/session_telemetry.rs:1139`) but no cache-miss detector → [[no-cache-miss-detection]].
- **Product analytics**: separate `codex-analytics` client posting to `{base_url}/codex/analytics-events/events`, 10 s send timeout / 25 s flush, dedupe 4 096 keys (`codex-rs/analytics/src/client.rs:88-91`, `:154`); disabled by top-level `analytics` config (`codex-rs/config/src/config_toml.rs:530-532`).
- **Trace fan-out**: `codex-rs/otel-trace-websocket` forwards loopback OTLP trace batches to a WebSocket listener, best effort, lagging clients lose batches (`codex-rs/otel-trace-websocket/src/lib.rs:1-5`).
- **Feedback**: `codex-rs/feedback` keeps an in-memory log ring (`DEFAULT_MAX_BYTES` 4 MiB) and uploads via Sentry with attachments (doctor report, apps caches, windows-sandbox log, rollout); rate limit 60 s, part upload timeout 300 s, daemon logs ≤ 256 KiB (`codex-rs/feedback/src/lib.rs:53-60`; `codex-rs/feedback/Cargo.toml:22`; `codex-rs/feedback/src/upload.rs:25`; `codex-rs/feedback/src/report_upload.rs:28`; `codex-rs/feedback/src/daemon_logs.rs:12`). App-server `feedback/upload`, TUI `/feedback`; `feedback = false` disables.
- **Install/environment identity**: `codex-rs/install-context` classifies install method Standalone/Npm/Bun/Brew (`codex-rs/install-context/src/lib.rs:55-77`) for update prompts (npm launcher sets `CODEX_MANAGED_BY_NPM`/`CODEX_MANAGED_BY_BUN`, [[codex--harness-package-distribution|harness-package-distribution]]); `codex-rs/build-info` resolves version/commit/target; `codex-rs/terminal-detection` feeds terminal metadata to the OTEL user-agent and TUI and "must not execute helpers selected by an untrusted PATH" (`codex-rs/terminal-detection/src/lib.rs:1-5`); `codex-rs/diagnostics` provides process-wide self-registering gauges (e.g. `core.mailbox.pending`).
- **Request attribution**: per-request metadata headers → [[request-attribution-metadata]].

## Constants
| name | value | path:line |
|---|---|---|
| default metrics exporter | Statsig (OTLP/HTTP) | codex-rs/config/src/types.rs:670-682 |
| analytics send / flush timeout | 10 s / 25 s | codex-rs/analytics/src/client.rs:88-90 |
| analytics dedupe keys | 4 096 | codex-rs/analytics/src/client.rs:154 |
| feedback log ring | 4 MiB | codex-rs/feedback/src/lib.rs:60 |
| feedback rate limit / part upload timeout | 60 s / 300 s | codex-rs/feedback/src/lib.rs:53-60; codex-rs/feedback/src/upload.rs:25 |
| agent-response / guardian log caps | 64 KiB | codex-rs/otel/src/agent_response.rs:18; codex-rs/otel/src/guardian_assessment.rs:13 |

## Quirks
- Terminal detection was hardened after repo-controlled PATH could make startup/doctor run workspace helpers (`637c3227b3` 2026-09-02) → [[untrusted-repo-loads-executable-config]].

## Versus pi
- [[pi--install-telemetry]]: pi sends only an opt-out version ping after installs/updates and ships an unused vendor-neutral tracing contract; codex ships full OTEL (metrics on by default to Statsig), a separate product-analytics client, and Sentry feedback upload with logs and rollout attachments.
