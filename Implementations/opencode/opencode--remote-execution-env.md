---
type: implementation
harness: opencode
concept: remote-execution-env
commit: ecc4916b5a
files: [packages/opencode/src/control-plane/types.ts:28-40, packages/opencode/src/control-plane/adapters/index.ts:1-20, packages/opencode/src/plugin/index.ts:158-161, packages/opencode/src/control-plane/workspace.ts:184-200, packages/opencode/src/control-plane/workspace.ts:325-335, packages/opencode/src/control-plane/workspace.ts:395-410, packages/opencode/src/control-plane/workspace.ts:529-536, packages/opencode/src/control-plane/workspace.ts:559-620, packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:60-87, packages/opencode/src/server/proxy-util.ts:15-18]
---
[[remote-execution-env]] in [[opencode]].

## Mechanism
### Legacy runtime (control-plane workspaces, `OPENCODE_EXPERIMENTAL_WORKSPACES`)
- **Whole remote agent, not a tool daemon**: a workspace target is either `{type: "local", directory}` or `{type: "remote", url, headers?}` (`packages/opencode/src/control-plane/types.ts:28-40`); a remote target is another full opencode server.
- **Adapters**: built-in `worktree` adapter only ([[git-worktree-isolation]]) (`packages/opencode/src/control-plane/adapters/index.ts:1-20`); plugins register more via `experimental_workspace.register(type, adapter)` (`packages/opencode/src/plugin/index.ts:158-161`). Example "folder" adapter and a dev "debug" adapter that simulates a remote by running a second server. opencode ships no VM/container adapter.
- **Provisioning env**: adapter `create` receives `OPENCODE_AUTH_CONTENT` (all stored credentials), `OPENCODE_WORKSPACE_ID`, `OPENCODE_EXPERIMENTAL_WORKSPACES=true` and OTEL vars (`packages/opencode/src/control-plane/workspace.ts:529-536`) ([[credentials-forwarded-to-execution-target]]).
- **Routing**: instance middleware resolves the workspace from the session's `workspace_id` or `?workspace=`; remote targets are HTTP-proxied with `x-opencode-directory`/`x-opencode-workspace` stripped (`packages/opencode/src/server/routes/instance/httpapi/middleware/workspace-routing.ts:60-87`; `packages/opencode/src/server/proxy-util.ts:15-18`).
- **Replication**: per workspace a fiber opens SSE `GET <remote>/global/event` (`packages/opencode/src/control-plane/workspace.ts:184-200`), POSTs `/sync/history` to backfill (`packages/opencode/src/control-plane/workspace.ts:325-335`), and replays each `sync` event into the local event store with `ownerID: workspace.id` (`packages/opencode/src/control-plane/workspace.ts:395-410`), so local projections mirror remote sessions ([[event-sourced-session-store]]).
- **Session warp** (`packages/opencode/src/control-plane/workspace.ts:559-620`): flush history from a remote source (or cancel a local run), `events.claim(sessionID, newOwner)` so later events from the old owner are ignored, and with `copyChanges` fetch the VCS diff (`vcs.diffRaw` or `/vcs/diff/raw`) and apply it in the target (`vcs.apply` or `/vcs/apply`).
- **Isolation**: directory-level (worktree) or host-level (whatever the adapter provisions); no process sandbox (`SECURITY.md:15-19`).

### v2 runtime
- `packages/core/src/control-plane/move-session.ts` and `workspace.sql.ts` hold the v2 port of session move and the table (not read in detail). Location `workspaceID` is "reserved for future placement semantics" (`specs/v2/session.md:48`).

## Constants
| name | value | path:line |
|---|---|---|
| control-plane request timeout | `TIMEOUT = 5000` | `packages/opencode/src/control-plane/workspace.ts:890` |

## Evolution
- 2026-02-27 `c12ce2ffff` basic remote workspace support (#15120).
- 2026-03-03 `7f37acdaaa` workspace integration and adapter interface reworked (#15895).
- 2026-05-04 `22a4a9df8b` session warping (#25768).

## Quirks / drift
- Remote execution moves the whole agent (model calls included) and its credentials, the opposite of tool-only isolation ([[tool-only-isolation]]).

pi contrast: pi-env keeps the agent local and runs only fs/exec on the remote through a small Rust daemon over pinned SSH ([[pi--remote-execution-env|pi]]).
