---
type: implementation
harness: opencode
concept: git-worktree-isolation
commit: ecc4916b5a
files: [packages/opencode/src/worktree/index.ts:23-36, packages/opencode/src/worktree/index.ts:119-128, packages/opencode/src/worktree/index.ts:183-225, packages/opencode/src/project/project.ts:55, packages/opencode/src/project/project.ts:97, packages/opencode/src/control-plane/adapters/worktree.ts:89-92]
---
[[git-worktree-isolation]] in [[opencode]].

## Mechanism
### Legacy runtime
- `Worktree.Service`: create, list, remove, reset (`packages/opencode/src/worktree/index.ts:119-128`).
- **Create** (`packages/opencode/src/worktree/index.ts:183-225`): generate a unique name (checks `refs/heads/opencode/<name>` with `git show-ref`); root `<data>/worktree/<projectID>`; `git worktree add --no-checkout -b opencode/<name> <dir>` or `--detach <dir> HEAD`; optional `startCommand` runs after creation (schema `packages/opencode/src/worktree/index.ts:32`).
- **Bookkeeping**: the project row keeps a `sandboxes` list of worktree directories (`packages/opencode/src/project/project.ts:55`, `packages/opencode/src/project/project.ts:97`). In opencode code "sandbox" means a worktree, not process isolation.
- **Workspaces**: the built-in `worktree` control-plane adapter returns `{type: "local", directory}` (`packages/opencode/src/control-plane/adapters/worktree.ts:89-92`), so a worktree is a local workspace target; sessions can be warped between workspaces with `copyChanges` ([[remote-execution-env]]).
- Requests select the worktree by `?directory=`; each gets its own instance state ([[location-scoped-runtime]]).

### v2 runtime
- Location `workspaceID` exists but placement semantics are reserved (`specs/v2/session.md:48`).

## Constants
| name | value | path:line |
|---|---|---|
| branch prefix | `opencode/` | `packages/opencode/src/worktree/index.ts:183` |
| root | `<data>/worktree/<projectID>` | `packages/opencode/src/worktree/index.ts:208` |

## Evolution
- 2026-01-03 `0b4af95223` "add sandbox support for git worktrees to allow working in multiple directories per project".
- 2026-02-27 `c12ce2ffff` / 2026-03-03 `7f37acdaaa` worktree became one workspace adapter.

## Quirks / drift
- Created with `--no-checkout`, then populated by `git reset --hard` inside the worktree during boot, before the start command (`packages/opencode/src/worktree/index.ts:231-240`).

pi contrast: no harness worktrees; parallel-agent git rules live in `AGENTS.md` ([[shared-worktree-agents-clobber-each-other]]).
