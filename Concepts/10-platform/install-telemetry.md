---
type: concept
stage: architecture
tier: candidate
aliases: ["enableInstallTelemetry", "PI_TELEMETRY", "report-install", "pi-telemetry", "TelemetryContext", "provider attribution headers", "enableAnalytics", "opt-out-install-telemetry", "vendor-neutral-telemetry-contract", "typed-telemetry-schema", OTEL_EXPORTER_OTLP_ENDPOINT, experimental.openTelemetry]
harnesses: [pi, opencode]
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
- **None + opt-in tracing**: no analytics client or install ping found; OTLP export only when the endpoint env var is set; AI SDK spans behind a config flag (opencode).

## Implementations
- [[pi--install-telemetry|pi]] — opt-out version ping to pi.dev after updates, `PI_TELEMETRY`/`PI_OFFLINE`, attribution headers gated by same flag; `@earendil-works/pi-telemetry` contract wired only as an unused `telemetryContext` option.
- [[opencode--install-telemetry|opencode]] — no product analytics found in `packages/opencode`/`packages/core`; opt-in OTLP and AI SDK spans tagged `session.id`.

## Failures
- [[telemetry-docs-outlive-code]]

## Related
[[layered-settings]] · [[harness-evals]] · [[session-export-share]] · [[unified-provider-api]] · [[http-transport-hardening]]
