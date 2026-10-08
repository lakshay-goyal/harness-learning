---
type: implementation
harness: opencode
concept: permission-ruleset
commit: ecc4916b5a
files: [packages/opencode/src/permission/index.ts:28-38, packages/opencode/src/permission/index.ts:67-167, packages/opencode/src/permission/index.ts:186-219, packages/core/src/util/wildcard.ts:3-14, packages/opencode/src/agent/agent.ts:119-135, packages/opencode/src/session/processor.ts:200-202, packages/opencode/src/session/processor.ts:647, packages/opencode/src/cli/cmd/run.ts:242-256, packages/opencode/src/cli/cmd/run.ts:801-820, packages/core/src/v1/permission.ts:7-26, packages/core/src/permission.ts:15, packages/core/src/permission.ts:76-290, packages/core/src/plugin/agent.ts:97-118, packages/core/src/policy.ts:36-42, SECURITY.md:15-19]
---
[[permission-ruleset]] in [[opencode]].

## Mechanism
### Legacy runtime (`packages/opencode/src/**`)
- **Rule** = `{permission, pattern, action: allow|ask|deny}`. `evaluate(permission, pattern, ...rulesets)` = `rulesets.flat().findLast(...)` where both permission and pattern match; no match → `ask` (`packages/opencode/src/permission/index.ts:28-38`).
- **Wildcard** (`packages/core/src/util/wildcard.ts:3-14`): `*` → `.*` (crosses `/`, so `*.env` matches nested paths), `?` → `.`, trailing ` *` becomes optional `( .*)?` so `git status *` also matches bare `git status` (`62702fbd11`). Backslashes normalized; case-insensitive on Windows.
- **Config forms**: `"edit": "ask"` or `{pattern: action}`; `~`/`$HOME` expanded (`packages/opencode/src/permission/index.ts:178-198`; `2a370f8038`, `fd9ee435ec`). Env `OPENCODE_PERMISSION` JSON deep-merged last (`packages/opencode/src/config/config.ts:559-561`).
- **Layering per agent**: `merge(defaults, agent-specific, user)` = plain concatenation, so user config always wins (`packages/opencode/src/agent/agent.ts:141-152`). Session rules (subagent denies, prompt-level `tools` map, `packages/opencode/src/session/prompt.ts:1059-1067`) are concatenated after the agent's at request time (`packages/opencode/src/session/llm/request.ts:210-216`).
- **Defaults** (`packages/opencode/src/agent/agent.ts:119-135`): `*: allow`, `doom_loop: ask`, `external_directory: {*: ask, truncation dir / tmp / skill dirs / reference dirs: allow}`, `question`/`plan_enter`/`plan_exit: deny`, `read: {*: allow, *.env: ask, *.env.*: ask, *.env.example: allow}` ("mirrors github.com/github/gitignore Node.gitignore").
- **Tool visibility**: `disabled()` hides a tool when the last rule for its permission has pattern `*` and action deny; `edit`/`write`/`apply_patch` share permission `edit`, MCP resource tools share `read` (`packages/opencode/src/permission/index.ts:204-219`).
- **ask** (`packages/opencode/src/permission/index.ts:67-107`): any pattern evaluating to deny → `DeniedError` immediately (message lists matching rules as JSON, `packages/core/src/v1/permission.ts:22-26`); all allow → return; else create pending request, publish `permission.asked`, await a Deferred.
- **reply** (`packages/opencode/src/permission/index.ts:109-167`):
  - `reject` → `RejectedError`, or `CorrectedError{feedback}` if a message was given; then **every other pending request in the same session is rejected** too.
  - `once` → resolve.
  - `always` → push `{permission, pattern, allow}` for the request's `always` patterns into the **instance-level in-memory** `approved` list (shared by all sessions of the directory, lost on restart), then auto-resolve other pending requests of that session that now evaluate to allow.
- **"always" scope per tool**: edit/read/todowrite/task use `["*"]` (one "always" on a `.env` read allows every read); bash uses the arity prefix ([[shell-command-permission-parsing]]); skill uses the skill name; `external_directory` uses `<dir>/*`.
- **Model-facing text** (`packages/core/src/v1/permission.ts:7-26`): "The user rejected permission to use this specific tool call." / "...with the following feedback: …" / "The user has specified a rule which prevents you from using this specific tool call. Here are some of the relevant rules …".
- **Effect on the loop**: a plain `RejectedError` (or rejected question) sets `ctx.blocked = ctx.shouldBreak` (`packages/opencode/src/session/processor.ts:200-202`); `shouldBreak` is true unless `experimental.continue_loop_on_deny` (`packages/opencode/src/session/processor.ts:647`); processor returns `stop` (`packages/opencode/src/session/processor.ts:694`). `CorrectedError` and rule-based `DeniedError` are ordinary tool errors; the loop continues.
- **Headless `opencode run`** (`packages/opencode/src/cli/cmd/run.ts`): non-interactive sessions add `question`/`plan_enter`/`plan_exit: deny` (`packages/opencode/src/cli/cmd/run.ts:430-447`); each `permission.asked` for a tracked session (root and subagents, `08faeb3893`) is auto-rejected with a printed warning, or replied `once` with `--auto` (hidden aliases `--yolo`, `--dangerously-skip-permissions`) (`packages/opencode/src/cli/cmd/run.ts:242-256`, `packages/opencode/src/cli/cmd/run.ts:801-820`). Explicit denies still deny.
- **Clients**: asks are events; TUI, web app, ACP (`requestPermission` with `allow_once`/`allow_always`/`reject_once`, fail closed when absent, `packages/opencode/src/acp/permission.ts:21-23,56-62`) answer via the HTTP `permission` routes.

### v2 runtime (`packages/core/src/**`)
- Rule = `{action, resource, effect}`; same `findLast` evaluate, default `ask` (`packages/core/src/permission.ts:76-86`).
- **Evaluation order** (`packages/core/src/permission.ts:155-162`): configured agent rules first; if any resource is denied → `BlockedError`. Otherwise evaluate `[...configured, ...saved]`. Saved approvals therefore can never override a configured deny.
- **Saved approvals**: `always` with a `save` list persists rules per project in SQL (`PermissionSaved`, `packages/core/src/permission/saved.ts`; `packages/core/src/permission.ts:251-257`), then auto-approves other now-allowed pending requests in any session (`packages/core/src/permission.ts:261-285`). Reject cascades to the session's other pending requests (`packages/core/src/permission.ts:230-247`).
- **Missing agent** → `[*:*:deny]` (`packages/core/src/permission.ts:15`). Agent-less sessions evaluate as the default `build` agent (`specs/v2/session.md:191`).
- Pending requests keep the agent of the provider turn that issued the call; a later agent switch cannot change it (`CONTEXT.md:125`).
- **Decline**: `assert` turns `DeclinedError` into a defect (`packages/core/src/permission.ts:209`), which ends the run (`709af58612`). `CorrectedError`/`BlockedError` are typed errors, but leaf tools map every error to a generic `ToolFailure` (e.g. "Unable to execute command: <cmd>", `packages/core/src/tool/bash.ts:196`), so the feedback text never reaches the model ([[permission-feedback-lost-in-tool-error]]).
- **Build defaults** (`packages/core/src/plugin/agent.ts:102-119`): `{action: "*", resource: "*", effect: "allow"}`, `external_directory` ask except truncation dir + tmp, `question`/`plan_enter`/`plan_exit` deny, `.env` reads ask. No `doom_loop` rule exists in v2 (grep `packages/core/src` finds only the v1 schema).
- **Policy** (separate, no `ask`): ordered `{effect, action, resource}` statements, `findLast` with caller fallback (`packages/core/src/policy.ts:36-42`), used for `provider.use` (`packages/core/src/catalog.ts:163`); user-global beats repository (`specs/v2/provider-policy.md:154-202`).

## Constants
| name | value | path:line |
|---|---|---|
| no-match default | `ask` | `packages/opencode/src/permission/index.ts:32-36`; `packages/core/src/permission.ts:80-84` |
| build default | `*: allow` | `packages/opencode/src/agent/agent.ts:120`; `packages/core/src/plugin/agent.ts:109` |
| `doom_loop` default | `ask` | `packages/opencode/src/agent/agent.ts:121` |
| `.env` reads | `*.env`/`*.env.*` ask, `*.env.example` allow | `packages/opencode/src/agent/agent.ts:129-134` |
| missing-agent ruleset | `*:*:deny` | `packages/core/src/permission.ts:15` |

## Evolution
- 2025-10-31 `a3ba740de4` headless `run` hung on permission prompts → responder added.
- 2025-11-09 `4e549b1c05` `doom_loop` and `external_directory` became user-configurable permissions (#4095).
- 2025-12-15 `7368342bab` `experimental.continue_loop_on_deny`.
- 2026-01-01 `351ddeed91` "Permission rework" (#6319): pattern rules, arity prefixes.
- 2026-01-04 `3611260405` hardcoded `.env` read block replaced by default rules.
- 2026-01-12 `62702fbd11` trailing ` *` optional.
- 2026-03-14 `2fc06c5a17` legacy permission module deleted; `permission.ask` plugin hook lost its trigger ([[dead-hook-in-public-api]]).
- 2026-04-25 `66f93035b0`, `a9740b9133`; 2026-05-12 `65368f609d` rule order preserved via layered arrays ([[permission-rule-order-lost-in-object-config]]).
- 2026-06-01 `9b815bcbd2` v2 location-based permission service. 2026-06-06 `4814ab3a3d` v2 tool definitions filtered by agent permissions. 2026-07-04 `709af58612` v2 stops after declined permissions ([[declined-action-retried]]).
- 2026-08-20 `08faeb3893` `run` answers subagent permission asks too.

## Quirks / drift
- Legacy `approved` is appended after the configured ruleset (`packages/opencode/src/permission/index.ts:73`), so an earlier "always" for `edit:*` would override a later configured or agent `edit` deny in the same instance, e.g. plan's (inferred from code, unverified at runtime). v2 fixed this ordering.
- `explore` puts `*: deny` after the defaults, so its `doom_loop` evaluates to deny, not ask (`packages/opencode/src/agent/agent.ts:196-209`, inferred).
- `SECURITY.md:15-19`: "The permission system exists as a UX feature … it is not designed to provide security isolation."
- v2 removes `experimental.continue_loop_on_deny` and the boolean `tools` map (`specs/v2/config.md:292`).

pi contrast: no rules and no prompts at all ([[no-permission-prompts]]); policy is an opt-in `tool_call` hook ([[pi--tool-call-gate|pi]]).
