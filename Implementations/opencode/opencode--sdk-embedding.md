---
type: implementation
harness: opencode
concept: sdk-embedding
commit: ecc4916b5a
files: [packages/sdk/js/script/build.ts:10-16, packages/sdk/js/src/index.ts:8-20, packages/sdk/js/src/server.ts:22-40, packages/sdk-next/src/index.ts:1-16, CONTEXT.md:160-164]
---
[[sdk-embedding]] in [[opencode]].

## Mechanism
### Legacy runtime (`@opencode-ai/sdk`)
- **Not in-process**: the SDK is an HTTP client generated from the server's OpenAPI (`bun dev generate > openapi.json`, then `@hey-api/openapi-ts`) (`packages/sdk/js/script/build.ts:10-16`).
- `createOpencode(options)` = `createOpencodeServer` (spawns the `opencode serve --hostname=127.0.0.1 --port=4096` CLI via `cross-spawn`, parses the URL from stdout) + `createOpencodeClient({baseUrl})` → `{client, server}` (`packages/sdk/js/src/index.ts:8-20`; `packages/sdk/js/src/server.ts:22-40`).
- Same client powers the TUI (over worker RPC), ACP bridge, `opencode run`, GitHub agent and plugins (`input.client` in the plugin context) ([[client-server-session-split]]).
- Embedding = process boundary + HTTP + SSE; injection points are only what the HTTP API and config expose.

### v2 runtime (`@opencode-ai/sdk-next`, `@opencode-ai/client`)
- **Embedded OpenCode**: the Effect-native SDK "executes Server's assembled `HttpRouter` in memory. It opens no listener and performs no network I/O, while preserving Server routing, middleware, codecs, handlers, and errors" (`CONTEXT.md:162`).
- Networked and embedded clients share one public `HttpApi`; embedded-only same-process capabilities extend Embedded OpenCode separately (`CONTEXT.md:164`). Promise and Effect clients ship from `@opencode-ai/client`; `sdk-next` will take over the `@opencode-ai/sdk` name (`CONTEXT.md:160-161`).
- `packages/sdk-next/src/index.ts:1-16` re-exports `OpenCode`, `Tool` and the client's Effect schema facade.

## Constants
| name | value | path:line |
|---|---|---|
| SDK-spawned server | `127.0.0.1:4096` | `packages/sdk/js/src/server.ts:25-26` |

## Evolution
- 2026-06-03 `76ee87ead8` embedded v2 session runtime (branch `feat/opencode-embedded-api`, `specs/v2/schema-changelog.md:41`).
- 2026-06-25 `cdd67cf30f` "add HttpApi clients and embedded host" (#33445).

## Quirks / drift
- The shipping SDK always costs a child process and a TCP port; in-process embedding exists only in the v2 packages.

pi contrast: pi's SDK is in-process from the start (`createAgentSession`, injectable services) ([[pi--sdk-embedding|pi]]).
