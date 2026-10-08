---
type: implementation
harness: opencode
concept: workspace-boundary-check
commit: ecc4916b5a
files: [packages/opencode/src/tool/external-directory.ts:15-46, packages/opencode/src/project/instance-context.ts:14-24, packages/core/src/fs-util.ts:270-273, packages/opencode/src/agent/agent.ts:108-125, packages/core/src/location-mutation.ts:80-105, specs/v2/schema-changelog.md:253-269]
---
[[workspace-boundary-check]] in [[opencode]].

## Mechanism
### Legacy runtime
- `assertExternalDirectoryEffect(ctx, target, {bypass, kind})` (`packages/opencode/src/tool/external-directory.ts:15-46`): returns early when inside the project; else asks `external_directory` with pattern and "always" = `<dir>/*` (dir = target for directories, parent for files), metadata `{filepath, parentDir}`.
- Called by read, edit, write, apply_patch (including move targets), grep, glob, and the shell scan for file-command path args ([[shell-command-permission-parsing]]).
- `containsPath` (`packages/opencode/src/project/instance-context.ts:18-24`): inside `ctx.directory` OR `ctx.worktree`; worktree check skipped when worktree is `/` (non-git projects, "would match ANY absolute path").
- `FSUtil.contains` is lexical: `path.relative(parent, child)` not absolute and not `..`-prefixed (`packages/core/src/fs-util.ts:270-273`). No `realpath`, so an in-project symlink to an outside dir passes as inside (observed from code; exploit unverified).
- Defaults: `external_directory: {"*": "ask"}` with whitelisted harness dirs allowed: truncation spill glob, `<tmp>/*`, every skill dir, every configured reference dir (`packages/opencode/src/agent/agent.ts:108-125`). Every agent is force-allowed the truncation dir unless it explicitly denies it (`packages/opencode/src/agent/agent.ts:296-309`). Plan agent also allows `<data>/plans/*`.
- Subagent sessions inherit the parent session's `external_directory` rules (`packages/opencode/src/agent/subagent-permissions.ts:21-23`).

### v2 runtime
- `LocationMutation` resolves the location root with `fs.realPath` and canonicalizes each target (existing path → `realPath`; new path → walk up to the nearest existing ancestor and canonicalize it) (`packages/core/src/location-mutation.ts:80-105`).
- Spec: relative mutation paths resolve within the Location; absolute external paths need explicit `external_directory` approval **before** the leaf approval; named references are read-only; "Revalidate path authority immediately before write mechanics" (`specs/v2/schema-changelog.md:253-269`). Reason given: "symlink/path-swap checks without pretending path APIs provide a syscall-level sandbox".
- v2 build defaults whitelist only truncation dir + tmp (`packages/core/src/plugin/agent.ts:99-106`).

## Constants
| name | value | path:line |
|---|---|---|
| default | `external_directory: {"*": "ask"}` | `packages/opencode/src/agent/agent.ts:122-125` |
| approval key | `<dir>/*` | `packages/opencode/src/tool/external-directory.ts:29-32` |

## Evolution
- 2025-11-09 `4e549b1c05` `external_directory` user-configurable.
- 2026-01-11 `fa79736b87` check against the worktree, not just cwd, for subdirectories.
- 2026-04-30 `d7701dbfb6` subagent sessions keep parent `external_directory` rules.
- 2026-05-09 `ba9e4b67ed` read permission matched absolute paths while edit/write matched worktree-relative → read uses worktree-relative.
- 2026-05-11 `1a28924ed8` grep evaluated the external-directory check wrongly.

## Quirks / drift
- Legacy lexical containment vs v2 realpath: the symlink gap exists only in the shipping runtime ([[read-path-traversal]] is the analogous pi failure).
- One "always" on `<dir>/*` allows every file in that directory for the rest of the instance.

pi contrast: no boundary at all; cwd is only a default ([[no-cwd-confinement]]).
