---
type: implementation
harness: pi
concept: install-telemetry
commit: b30a6dd77
files: [packages/coding-agent/src/core/telemetry.ts:3-13, packages/coding-agent/src/core/settings-manager.ts:155-156, packages/coding-agent/src/core/settings-manager.ts:1164-1190, packages/coding-agent/src/modes/interactive/interactive-mode.ts:1326-1367, packages/coding-agent/src/core/provider-attribution.ts:36-77, packages/telemetry/src/index.ts:1-354, packages/telemetry/src/testing/conformance.ts:61-315, packages/ai/src/types.ts:140-141]
---
[[install-telemetry]] in [[pi]].

## Mechanism
### A. Install ping (the actual phone-home)
- Trigger: fresh install (no `lastChangelogVersion`) or changelog-detected update → `reportInstallTelemetry(VERSION)` (`modes/interactive/interactive-mode.ts:1334-1345`). Skipped for resumed sessions with messages (changelog path skipped, `:1326-1328`).
- Transport: fire-and-forget `GET https://pi.dev/api/report-install?version=<v>`, `User-Agent: pi/<ver> (<platform>; node|bun/<ver>; <arch>)` (`utils/pi-user-agent.ts`), 5 s timeout, errors swallowed (`interactive-mode.ts:1351-1367`). Skipped when `PI_OFFLINE` set (`:1352-1354`).
- Opt-OUT: `enableInstallTelemetry` default **true** (`core/settings-manager.ts:155,1164-1166`); `PI_TELEMETRY` env overrides — truthy enables, anything else disables (`core/telemetry.ts:3-13`; help `cli/args.ts:466`); `/settings` toggle "Install telemetry" (`components/settings-selector.ts:561-567`).
- Same switch gates provider attribution headers (`core/provider-attribution.ts:36-65`): OpenRouter `HTTP-Referer: https://pi.dev`, `X-OpenRouter-Title: pi`, `X-OpenRouter-Categories: cli-agent`; NVIDIA `X-BILLING-INVOKE-ORIGIN: Pi`; Cloudflare `User-Agent: pi-coding-agent`. OpenCode `x-opencode-session`/`x-opencode-client: pi` headers sent regardless (`:67-77`).
- Analytics: `enableAnalytics` default **false** (`settings-manager.ts:156,1174-1176`); first opt-in mints `trackingId = randomUUID()` (`:1183-1190`); offered in experimental first-run setup ("Share anonymous usage data" / "Don't share", initial highlight = share, `first-time-setup.ts:25-36`; `packages/coding-agent/src/main.ts:674-675`); copy promises `/privacy` (`first-time-setup.ts:77`) but at HEAD there is no consumer of `getEnableAnalytics()`/`getTrackingId()` and no `/privacy` command (grep). Bug reports strip `trackingId`/`deviceId` (`core/bug-report.ts:62`).
- Related outbound: `/bug` uploads a redacted bundle (settings/model URLs redacted, optional `session.jsonl`) to Radius `/v1/bug-reports` (`core/bug-report.ts:19-63,252-265`; `bug-report-upload.ts:12-37`; `3c75b2747`); local crash log keeps last 5 records / 7 days in `~/.pi/agent/crashes.json` (`core/crash-log.ts:5-22`); update check against pi.dev (`c745efc0d`).

### B. `@earendil-works/pi-telemetry` — vendor-neutral tracing contract
- Callback-based `TelemetryContext.startSpan(options, callback)`; span methods `addEvent`, `setAttributes`, `setStatus`; a span is a context for children (`packages/telemetry/src/index.ts:14-22`). `AttributeValue` = primitive scalars or homogeneous arrays (`:1`); `SpanStatus` = `ok | error{name,message}` (`:12`). No exporter, no global current span, no backend dep (`packages/telemetry/README.md:5-11`); adapt to OTel/Sentry/logs (`:13`); no `AsyncLocalStorage` → portable to browsers/workers (`packages/telemetry/README.md:391`).
- Schema DSL `TelemetrySchemaDefinition {version, spans}`; per span `parents` (`any | root_or_external | spans[…]`), `startAttributes` (with `required`), optional `endAttributes`, `events`, `status {default:"ok", errorWhen}`; attribute metadata `description`, `sensitive?`, `cardinality: low|high`, closed `values`, `examples` (`index.ts:26-74,28-42`). `createTypedSpanStarter(ctx, schemas)`: duplicate span names across schemas = type error (`:281-297,349-354`); **no runtime validation** (`:345-348`).
- `NOOP_TELEMETRY_CONTEXT` frozen no-op that still invokes the callback (`src/noop.ts:3-20`); `InMemoryTelemetryContext` reference adapter: detached deep copies, children of settled parent → noop, recording failures swallowed, throw → automatic `error` unless status set explicitly, unbounded process-local storage (`src/memory.ts:16-218`; `packages/telemetry/README.md:176`).
- Adapter contract (`packages/telemetry/README.md:120-130`): invoke callback synchronously exactly once; preserve return/rejection identity; span open until promise settles; last-write-wins status; ignore `undefined` attrs; recording synchronous, passive, non-throwing; post-settlement calls ignored; failed recordings atomic.
- Conformance suite `@earendil-works/pi-telemetry/testing` → `createTelemetryAdapterConformance(factory)` groups: callback lifecycle, status, recording (atomic failed `setAttributes`), parentage, passivity (Proxy that throws on every trap simulates unreadable inputs) (`src/testing/conformance.ts:43-315`) → [[pi--harness-evals]].
- Privacy guidance: never persist telemetry in records/snapshots; avoid prompts, completions, tool args/output, file contents, payloads, headers, credentials unless schema allows (`packages/telemetry/README.md:387-389`).
- Wiring at HEAD: only `pi-ai` depends on it, adding `telemetryContext?: TelemetryContext` to `ProviderRequestOptions` (`packages/ai/src/types.ts:1,140-141`) threaded via `buildBaseOptions` (`ai/src/api/simple-options.ts:48`); **no span is emitted anywhere in production code** (grep `startSpan|pi.harness.|pi.ai.` empty).
- Deleted schema (`git show 7fd478a2e^:packages/agent/src/harness/telemetry.ts`): spans `pi.ai.request`, `pi.harness.run/.compaction/.navigation/.checkpoint/.turn/.step/.tool/.hook/.sleep/.event_handler`, `pi.session.write`; attributes `pi.ai.usage.{input,output,cache_read,cache_write,reasoning,total}_tokens`, `pi.ai.usage.cost`, `pi.ai.stream.time_to_first_chunk_ms`, `pi.ai.http.status_code`.

## Constants
| name | value | path:line |
|---|---|---|
| `enableInstallTelemetry` | true | `settings-manager.ts:155` |
| `enableAnalytics` | false | `settings-manager.ts:156` |
| install ping timeout | 5 s | `interactive-mode.ts:1351-1367` |
| crash log retention | 5 records / 7 days | `core/crash-log.ts:5-22` |
| version check timeout | 10 000 ms | `utils/version-check.ts:6` |

## Evolution
- 2026-04-14 `7371c30c0` install telemetry ping controls; 2026-04-20 `62c1c4031` OpenRouter attribution headers (+`telemetry.ts`); 2026-04-28 `c745efc0d` update check against pi.dev (#3877); 2026-04-30 `904b843fe` docs clarify telemetry/update checks; 2026-06-02 `601480122` NVIDIA attribution.
- 2026-08-05 `04d6447f7` typed telemetry contracts in pi-ai + agent harness; `6b461b75b` extracted into `packages/telemetry`, ai keeps only `telemetryContext`; 2026-08-06 `35f5c265d` in-memory adapter, noop, conformance suite, typed multi-schema starters (telemetry v0.84.0).
- 2026-09-19 `3c75b2747` bug reports via Radius gateway.
- 2026-10-01 `7fd478a2e` harness telemetry schemas + pi-telemetry re-exports deleted with the old in-agent harness; README still advertises them.

## Evidence commits
`7371c30c0` `62c1c4031` `c745efc0d` `904b843fe` `601480122` `04d6447f7` `6b461b75b` `35f5c265d` `3c75b2747` `7fd478a2e`

## Quirks
- First-run dialog pre-highlights "share" while the setting defaults to false (UX intent unverified).
- Analytics setting and tracking id exist with no transport — promised `/privacy` command missing (open question).
- Two unrelated things are called "telemetry" (install ping vs tracing library).

## Failures
[[telemetry-docs-outlive-code]]
