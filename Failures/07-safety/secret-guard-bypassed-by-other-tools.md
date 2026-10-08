---
type: failure
concepts: [secret-handling, permission-ruleset, shell-command-permission-parsing]
harnesses: [opencode]
---
**Symptom** — (Observed from code, exploit not reproduced.) opencode asks before the `read` tool opens `.env` files, but the same bytes reach the model through other paths without that prompt: `bash` `cat .env` is evaluated only as a `bash` pattern under the default `*: allow`; `grep` runs ripgrep with `--hidden` (`packages/core/src/ripgrep.ts:224`) and is not checked against the `.env` rule (`.gitignore` usually, not always, excludes `.env`); one "always" on any read approves `read:*`, including later `.env` reads.

**Root cause** — The secret rule is keyed on one permission (`read: {*.env: ask, *.env.*: ask, *.env.example: allow}`, `packages/opencode/src/agent/agent.ts:128-134`) while shell and search tools carry their own permissions; the read tool's "always" scope is `["*"]`.

**Fix · [[opencode]]** — none; the hardcoded read-tool block was replaced by these rules in `3611260405` 2026-01-04 (and narrowed so `.envrc` is not blocked, `60db171b44` 2025-12-22). The permission system is declared UX, not security (`SECURITY.md:15-19`).

**Lesson** — A secret-file guard attached to one tool is advisory; any tool that can read the same bytes (shell, search, MCP) must be covered, or the guard must live below the tools (filesystem or sandbox).

Related: [[secret-handling]] · [[permission-ruleset]] · [[shell-command-permission-parsing]] · [[workspace-boundary-check]] · [[opencode--secret-handling|opencode]]
