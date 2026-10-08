---
type: concept
stage: permissions
tier: variant
aliases: [WritableRoot.read_only_subpaths, protected_metadata_names, PROTECTED_METADATA_PATH_NAMES, default_read_only_subpaths_for_writable_root, read-only .git within writable root, protected-path carve-outs]
harnesses: [codex]
---
Inside a writable sandbox root, metadata paths whose contents later run or are trusted with more authority than the agent (VCS hooks, agent config/rules, skills, credential-helper config) stay read-only — including first-time creation, renames of their ancestors, and pointer-resolved locations.

## Why
- "Writable workspace" naïvely includes `.git/hooks` (runs on the user's next `git commit`), the agent's own project config/rules (`.codex/` → auto-approved commands next session), skills (`.agents/`), and `~/.aws`-style profiles that select executable credential helpers. Writing any of these escalates out of the sandbox *later*, with no approval ([[agent-writes-its-own-escalation-config]]).
- Path carve-outs are easy to bypass: create the dir before the rule matches, rename an ancestor, follow a `.git` pointer file into another writable root ([[sandbox-path-binding-races]]).

## Design space
- **No carve-outs**: whole cwd writable — pi (no sandbox at all; [[no-cwd-confinement]]); pi example `protected-paths.ts` is a tool-name gate on `write`/`edit` only, not kernel-enforced ([[pi--tool-call-gate]]).
- **Fixed name list** at the top of each writable root (✔ codex: `.git`, `.agents`, `.codex`, `.aws`) vs. user-configurable deny-write globs (codex permission profiles can also express `none`/read entries).
- **Exists-only vs. always**: protect only if present (codex `.agents`, `.aws`) vs. protect even when missing so first creation needs approval (codex `.codex` at workspace root).
- **Pointer resolution**: also protect the resolved `gitdir:` of worktrees/submodules (✔ codex).
- **Rename protection** of ancestors (✔ codex Seatbelt) and late/literal path binding.
- **Enforcement**: kernel (Seatbelt `require-not`, bwrap `--ro-bind`, Windows deny ACEs) vs. tool-level checks.
- Out-of-workspace writes to the same dirs are a separate question ([[project-trust-gate]] decides whether project `.codex/` is even loaded).

## Implementations
- [[codex--protected-workspace-metadata|codex]] — `PROTECTED_METADATA_PATH_NAMES = [.git, .agents, .codex, .aws]`; `.git` dir/file + resolved gitdir; `.codex` protected even when missing; enforced in Seatbelt, bwrap, Windows ACLs.

## Failures
- [[agent-writes-its-own-escalation-config]]
- [[sandbox-path-binding-races]]

## Related
[[os-level-sandbox]] · [[project-trust-gate]] · [[command-rule-policy]] · [[skill-progressive-disclosure]] · [[layered-settings]] · [[no-cwd-confinement]] · [[secret-handling]]
