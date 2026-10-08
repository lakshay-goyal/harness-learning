---
type: implementation
harness: opencode
concept: install-telemetry
commit: ecc4916b5a
files: [packages/core/src/observability/otlp.ts:7-15, packages/opencode/src/session/llm.ts:208-222, packages/opencode/src/agent/agent.ts:376-393, packages/opencode/src/cli/upgrade.ts:8-40, packages/llm/DESIGN.md:852-873]
---
[[install-telemetry]] in [[opencode]].

## Mechanism
### Legacy runtime
- **No product analytics found**: grep for posthog/segment/analytics clients in `packages/opencode/src` and `packages/core/src` finds none (only a comment about gateway analytics in `packages/opencode/src/provider/provider.ts:923`). The console, web and desktop packages were not checked.
- **No install ping**: the startup `upgrade()` check fetches the latest version from the install channel (npm registry, GitHub releases, brew, …) to decide on updates ([[self-update]]); it sends no explicit telemetry payload (`packages/opencode/src/cli/upgrade.ts:8-40`).
- **Opt-in tracing**: OTLP exporter only when `OTEL_EXPORTER_OTLP_ENDPOINT` is set, headers from `OTEL_EXPORTER_OTLP_HEADERS` (`packages/core/src/observability/otlp.ts:7-15`). AI SDK telemetry enabled only with `experimental.openTelemetry`; spans tagged with `session.id`, metadata `userId: cfg.username ?? "unknown"` (`packages/opencode/src/session/llm.ts:208-222`; `packages/opencode/src/agent/agent.ts:388-393`).
- OTEL vars are forwarded to control-plane workspace targets with the credentials ([[secret-handling]]).

### v2 runtime
- Design draft: "Default telemetry records metadata only" (model identity, timing, usage, finish reasons, retries, tool names); "Prompts, model output, tool arguments, and tool results are never recorded by default" (`packages/llm/DESIGN.md:862-873`, status "Discussion draft").

## Constants
| name | value | path:line |
|---|---|---|
| OTLP enable | `OTEL_EXPORTER_OTLP_ENDPOINT` set | `packages/core/src/observability/otlp.ts:7` |

## Evolution
- Not mined (no telemetry-specific commits examined).

## Quirks / drift
- Share (`opncd.ai`) is the main data-leaving path besides the model provider, and it is opt-in per session ([[session-export-share]]).

pi contrast: opt-out anonymous version ping after updates, `PI_TELEMETRY` kill switch, vendor-neutral span contract ([[pi--install-telemetry|pi]]).
