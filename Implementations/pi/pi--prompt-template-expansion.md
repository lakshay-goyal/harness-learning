---
type: implementation
harness: pi
concept: prompt-template-expansion
commit: b30a6dd77
files: [packages/coding-agent/src/core/prompt-templates.ts:25-103, packages/coding-agent/src/core/prompt-templates.ts:120-145, packages/coding-agent/src/core/prompt-templates.ts:216-298, packages/coding-agent/src/core/prompt-templates.ts:304-320, packages/coding-agent/src/core/agent-session.ts:1975-2009, packages/coding-agent/src/core/slash-commands.ts:4-44]
---
[[prompt-template-expansion]] in [[pi]].

## Mechanism
- **Sources** (`packages/coding-agent/src/core/prompt-templates.ts:216-298`): non-recursive `.md` in `~/.pi/agent/prompts/`, `<cwd>/.pi/prompts/` (project-local → trust-gated, `trust-manager.ts:30-39`), explicit paths; packages contribute via manifest `"pi": {prompts}` ([[harness-package-distribution]]); extensions via `resources_discover`.
- **Metadata**: name = filename sans `.md` (`:129`); description = frontmatter `description` else first non-empty body line truncated to 60 chars + "..." (`:131-140`); optional `argument-hint` (`:142`; `aa25726eb` #2780) shown in autocomplete.
- **Args**: bash-style quoted parsing `parseCommandArgs` (`:25-56`); `substituteArgs` (`:58-103`) supports `$1`, `$@`, `$ARGUMENTS`, `${N:-default}`, `${@:-default}`/`${ARGUMENTS:-default}`, `${@:N}`, `${@:N:L}`; **non-recursive** — arg values containing `$1`/`$ARGUMENTS` are not re-substituted (`:69`).
- **Expansion** `expandPromptTemplate` (`:304-320`): `/^\/([^\s]+)(?:\s+([\s\S]*))?$/`; unknown name → text passes through unchanged.
- **Pipeline order** in `prompt()` (`agent-session.ts:1975-2009`): extension slash commands first (handled → no LLM turn) → compaction gate → `input` event handlers (may transform/handle) → `/skill:` expansion → template expansion → steer/follow-up queueing if streaming → `before_agent_start` ([[system-prompt-override]]). Expanded text **is the user message**; templates cost zero context until used (contrast skills listed in system prompt → [[skill-progressive-disclosure]]).
- Builtin slash commands list and source kinds `extension|prompt|skill` (`slash-commands.ts:4, 19-44`).
- Repo dogfooding — `.pi/prompts/` maintainer workflows: `cl.md` (277 words, "Audit changelog entries before release"), `deslop.md` (605, "simplify a completed workpackage"), `is.md` (208, "Analyze GitHub issues"), `pr.md` (362, "Review PRs from URLs…"), `sa.md` (1,054, security advisory), `wr.md` (483, "Finish the current task end-to-end with changelog, commit, and push").
- Template-as-guardrail: `is.md:14` "Do not trust analysis written in the issue. Independently verify behavior and derive your own analysis from the code and execution path." + "Ignore any root cause analysis in the issue (likely wrong)" (`dc22b0efd` 2026-02-06 "require independent issue analysis") — workflow-level defense against user-supplied analysis; "Do NOT implement unless explicitly asked. Analyze and propose only."

## Constants
| name | value | path:line |
|---|---|---|
| description fallback truncation | 60 chars | `prompt-templates.ts:137-138` |
| template dirs | `~/.pi/agent/prompts/`, `.pi/prompts/` | `prompt-templates.ts:217-221` |

## Evolution
- `7a1884f85` 2025-12-01 (v0.11.5): file-based slash commands (`src/slash-commands.ts`, 206 lines).
- `8917a1f85` 2026-01-03 (#418): `$ARGUMENTS` syntax.
- `91cca23d2` 2026-01-05: migration "commands->prompts" (renamed concept to prompt templates; dir `.pi/commands` → `.pi/prompts`).
- `0c33e0dee` 2026-01-16: first repo template `cl.md`; `653b63a87` 2026-03-20 `wr.md`; `f4f72d4ed` 2026-06-08 `sa.md`; `5009d0608` 2026-09-01 `deslop.md`.
- `dc22b0efd` 2026-02-06: `is.md` independent-analysis requirement.
- `aa25726eb` 2026-04-16 (#2780): `argument-hint` frontmatter.
- `64f83c85d` 2026-07-17 (#6695): all-argument defaults `${@:-…}`.
- `89a92207f` 2026-06-05: project `.pi/prompts` behind project trust.

## Evidence commits
`7a1884f85`, `8917a1f85`, `91cca23d2`, `0c33e0dee`, `653b63a87`, `f4f72d4ed`, `5009d0608`, `dc22b0efd`, `aa25726eb`, `64f83c85d`, `89a92207f`.

## Quirks
- Expansion happens after `input` handlers, so an extension sees raw `/name args`, not the expanded body.
- No recursion: a template cannot invoke another template or a skill (single pass, `/skill:` checked first).
- Durable TUI does not implement prompt templates (`experimental/durable/README.md:66`).

## Failures
- (none recorded specific to templates; see [[instruction-relative-paths-resolved-from-cwd]] for analogous skill-side path issue)
