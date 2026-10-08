---
type: implementation
harness: opencode
concept: path-normalization
commit: ecc4916b5a
files: [packages/opencode/src/tool/read.ts:235-236, packages/opencode/src/tool/edit.ts:80-82, packages/opencode/src/tool/write.ts:41-43, packages/opencode/src/tool/external-directory.ts:20-40, packages/opencode/src/project/instance-context.ts:18-24, packages/core/src/location-mutation.ts:26, packages/core/src/location-mutation.ts:84-103]
---
[[path-normalization]] in [[opencode]].

## Mechanism
### Legacy runtime
- Minimal normalization: relative → joined to `instance.directory`; absolute passed through (`packages/opencode/src/tool/read.ts:235-236`; `packages/opencode/src/tool/edit.ts:80-82`; `packages/opencode/src/tool/write.ts:41-43`). No `@`/`~`/unicode-space/`file://` handling found (grep; unverified for every tool).
- Windows: `FSUtil.normalizePath` before containment checks (`packages/opencode/src/tool/external-directory.ts:25`).
- Containment is **lexical**: `FSUtil.contains(directory|worktree, path)`; non-git projects (worktree `/`) skip the worktree check (`packages/opencode/src/project/instance-context.ts:18-24`). Outside → `external_directory` ask on `<dir>/*` (`external-directory.ts:26-40`) → [[workspace-boundary-check]].
- Missing read paths get sibling suggestions instead of normalization retries (`read.ts:94`).
### v2 runtime
- `LocationMutation.resolve`: `realPath` of the Location root and of the target (or nearest existing anchor), error reasons `relative_escape`, `location_escape`, `non_directory_ancestor` (`packages/core/src/location-mutation.ts:26,84-103`); symlink escapes rejected for reads (`specs/v2/session.md:197`).
- Admits it is not a syscall sandbox (`specs/v2/schema-changelog.md:270`) → [[no-sandbox]].

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- Legacy permission rework introducing `external_directory`: 2026-01-01 `351ddeed91` "Permission rework (#6319)".
- v2: canonical (realpath) containment from the first slice.

## Quirks / drift
- Legacy lexical check can be bypassed by an in-project symlink pointing outside (inference; v2 fixes it with `realPath`).

Failures: [[read-path-traversal]].

Contrast: pi normalizes model path quirks (`@`, `~`, unicode spaces, screenshot names) and deliberately has no confinement → [[pi--path-normalization|pi]], [[no-cwd-confinement]].
