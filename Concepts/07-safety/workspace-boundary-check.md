---
type: concept
stage: permissions
tier: candidate
aliases: [external_directory, assertExternalDirectory, assertExternalDirectoryEffect, containsPath, LocationMutation, "external-directory authorization"]
harnesses: [opencode]
---
File and shell path arguments that resolve outside the project or worktree trigger a separate `external_directory` permission.

## Why
- Without it, an allowed `edit` or `read` reaches `~/.ssh`, other repos or system files with the same approval as in-project work.
- A separate permission lets defaults stay permissive inside the project (`*: allow`) while asking at the boundary (opencode `external_directory: {"*": "ask"}`, `packages/opencode/src/agent/agent.ts:122-125`).
- Some outside dirs are the harness's own (tool-output spill dir, tmp, skill dirs, references) and must be whitelisted, or every truncated output read would prompt.

## Design space
- **None** (pi: cwd is a convenience, not a jail, [[no-cwd-confinement]]).
- **Containment test**: lexical `path.relative` (opencode legacy; a symlink inside the project pointing outside counts as inside) vs canonical `realPath` plus revalidation immediately before the write (opencode v2 `LocationMutation`).
- **Boundary**: cwd only vs cwd or git worktree (opencode legacy; worktree ignored for non-git projects where it is `/`).
- **Approval key**: parent directory glob `<dir>/*` (opencode), so "always" covers siblings.
- **Coverage**: file tools only vs also shell path arguments parsed from the command ([[shell-command-permission-parsing]]) vs syscall-level sandbox (not attempted; opencode v2 changelog: "without pretending path APIs provide a syscall-level sandbox").
- **Subagents**: inherit the parent session's `external_directory` rules (opencode).

## Implementations
- [[opencode--workspace-boundary-check|opencode]] — legacy `assertExternalDirectoryEffect` on read/edit/write/apply_patch/grep/glob + shell file-command args, lexical `containsPath`; v2 `LocationMutation` canonicalizes with `realPath` and requires `external_directory` before the leaf `edit` approval.

## Failures
- [[read-path-traversal]]
- [[secret-guard-bypassed-by-other-tools]]

## Tradeoffs
- [[cwd-confinement-vs-none]]

## Related
[[permission-ruleset]] · [[shell-command-permission-parsing]] · [[path-normalization]] · [[tool-only-isolation]] · [[git-worktree-isolation]] · [[no-cwd-confinement]] · [[no-sandbox]] · [[opencode]]
