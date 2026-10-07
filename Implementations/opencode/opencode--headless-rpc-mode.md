---
type: implementation
harness: opencode
concept: headless-rpc-mode
commit: ecc4916b5a
files: [packages/opencode/src/cli/cmd/run.ts:147-256, packages/opencode/src/cli/cmd/run.ts:430-447, packages/opencode/src/cli/cmd/run.ts:801-820, packages/opencode/src/cli/cmd/acp.ts:20-60, packages/opencode/src/acp/service.ts:97-128, packages/opencode/src/acp/permission.ts:18-62, packages/opencode/src/acp/permission.ts:102-117]
---
[[headless-rpc-mode]] in [[opencode]].

## Mechanism
### Legacy runtime
- **`opencode run` (one-shot)**: a client of the in-process or `--attach`ed server; options `--continue`, `--session`, `--share`, `--format default|json` ("raw JSON events"), `--command`, `--auto` (`packages/opencode/src/cli/cmd/run.ts:147-256`). JSON format prints server events as they arrive (`packages/opencode/src/cli/cmd/run.ts:679`).
- **Permission responder**: non-interactive sessions deny `question`/`plan_enter`/`plan_exit` (`packages/opencode/src/cli/cmd/run.ts:430-447`); every `permission.asked` for a session of this run (root or subagent) is auto-rejected with a warning, or replied `once` under `--auto` (`packages/opencode/src/cli/cmd/run.ts:801-820`) ([[permission-ruleset]], [[approval-wait-without-responder]]).
- **ACP (`opencode acp`)** — Agent Client Protocol over stdio NDJSON for editors:
  - Sets `OPENCODE_CLIENT=acp`, starts a real `Server.listen`, creates an SDK client with Basic-auth headers, and bridges stdin/stdout through `ndJsonStream` into `AgentSideConnection` (`packages/opencode/src/cli/cmd/acp.ts:20-60`). The ACP adapter is one more HTTP client.
  - `initialize`: `protocolVersion: 1`, `loadSession`, prompt `image` + `embeddedContext`, session `close`/`fork`/`list`/`resume`; auth method "Run `opencode auth login` in the terminal" (`packages/opencode/src/acp/service.ts:97-128`).
  - ACP modes = opencode agents with `mode !== "subagent"` and not hidden; editor-supplied `mcpServers` registered per directory via `sdk.mcp.add`.
  - Permissions: `permission.asked` → `requestPermission` with options `allow_once`/`allow_always`/`reject_once` → `once`/`always`/`reject`; missing capability or a thrown request → reject (`packages/opencode/src/acp/permission.ts:18-62`). Approved edits are pushed to the editor buffer with `writeTextFile` before opencode writes (`packages/opencode/src/acp/permission.ts:102-117`).
- **GitHub agent** is a third headless entry point with no permission responder ([[ci-agent-integration]]).

### v2 runtime
- No stdio mode of its own; headless consumers use the HTTP/embedded clients ([[sdk-embedding]]).

## Constants
| name | value | path:line |
|---|---|---|
| ACP protocol version | 1 | `packages/opencode/src/acp/service.ts:113` |

## Evolution
- 2025-07-31 `936f4cb0c6` permission state hangs.
- 2025-10-20 `f3f21194ae` ACP support (#2947).
- 2025-10-31 `a3ba740de4` headless run hung on permission prompts → responder.
- 2026-08-20 `08faeb3893` subagent permission asks answered in `run`.

## Quirks / drift
- Three headless entry points (`run`, ACP, GitHub) each implement their own permission handling; only GitHub has none.

pi contrast: pi's `--mode rpc` is a bidirectional JSONL command protocol with an extension-UI subprotocol, not an HTTP client ([[pi--headless-rpc-mode|pi]]).
