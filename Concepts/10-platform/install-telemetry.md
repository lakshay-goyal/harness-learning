---
type: concept
stage: architecture
tier: candidate
aliases: ["enableInstallTelemetry", "PI_TELEMETRY", "report-install", "pi-telemetry", "TelemetryContext", "provider attribution headers", "enableAnalytics", "opt-out-install-telemetry", "vendor-neutral-telemetry-contract", "typed-telemetry-schema", codex-otel, codex-analytics, codex-feedback, otel-trace-websocket, Statsig, "[otel]", install-context]
harnesses: [pi, codex]
---
What a harness sends home or exposes for observability: anonymous install/update pings and app-identifying headers (with kill switches), plus an in-process tracing contract the embedding app can adapt to its own backend.

## Why
- Maintainers want adoption counts; users want a clear, cheap off switch and offline mode.
- Gateways (OpenRouter, NVIDIA) attribute traffic by headers — same privacy switch should govern them.
- Library tracing without ambient state or a hard-wired exporter keeps the core portable (browser/workers) and avoids leaking prompts/tool output into telemetry.
- Docs for telemetry schemas rot when the code that emitted them is deleted ([[telemetry-docs-outlive-code]]).

## Design space
- **Default**: opt-in vs **opt-out** (pi install ping) vs none.
- **Payload**: version-only GET (pi) vs usage analytics with tracking id (pi `enableAnalytics`, default off; no transport at HEAD).
- **Trigger**: every start vs only after install/update (pi).
- **Kill switches**: setting + env (`PI_TELEMETRY`) + global offline (`PI_OFFLINE`) (pi).
- **Coupled outbound identity**: attribution headers under the same switch (pi) vs separate.
- **Tracing**: OTel dependency vs vendor-neutral callback-scoped span contract with no exporter, no global current span, typed schemas, conformance suite (pi `pi-telemetry`).
- **Schema enforcement**: runtime validation vs compile-time types only (pi).
- **Metrics on by default to vendor endpoint** (Statsig OTLP) with opt-in sensitive content logs ✔ codex.
- **User-initiated diagnostics upload** (log ring + rollout + doctor report via Sentry) ✔ codex.
- **Install-method detection** for update prompts (npm/bun/brew/standalone) ✔ codex.

## Implementations
- [[pi--install-telemetry|pi]] — opt-out version ping to pi.dev after updates, `PI_TELEMETRY`/`PI_OFFLINE`, attribution headers gated by same flag; `@earendil-works/pi-telemetry` contract wired only as an unused `telemetryContext` option.
- [[codex--install-telemetry|codex]] — OTEL with metrics exporter Statsig by default, opt-in prompt/response logging, separate product-analytics client (`analytics = false`), Sentry feedback upload with log ring + rollout attachments, install-method detection.

## Failures
- [[telemetry-docs-outlive-code]]

## Related
[[layered-settings]] · [[harness-evals]] · [[session-export-share]] · [[unified-provider-api]] · [[http-transport-hardening]]
