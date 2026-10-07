---
type: concept
stage: messages
tier: must-have
aliases: [prompt templates, "$ARGUMENTS", "$1", ".pi/prompts", slash commands, argument-hint, "!`cmd`", /init, /review, subtask, prompt_for_init_command.md, "/prompts:", custom prompts, "$CODEX_HOME/prompts"]
harnesses: [pi, opencode, codex]
---
User-side slash macros (markdown files) that expand, with positional arguments, into the user message before the turn starts.

## Why
- Repeatable workflows (issue triage, PR review, release wrap-up) otherwise get retyped or live permanently in the system prompt.
- Templates cost context only when invoked — unlike skills/rules that are always listed.
- A template can encode guardrails the system prompt shouldn't carry globally (pi `is.md`: "Do not trust analysis written in the issue").

## Design space
- **Markdown file per command, filename = name, bash-style args (`$1`, `$@`, `$ARGUMENTS`, defaults, slices), non-recursive substitution** (**pi chose**).
- Scripted commands with code (pi: extension commands, handled before templates).
- Templates vs skills: template = user-triggered text injection; skill = model-discoverable instructions ([[skill-progressive-disclosure]]).
- Scope: global + project (project trust-gated in pi) + packages.
- Built-in canned prompts shipped by the harness (codex `/init`: fixed Markdown file submitted as a normal user message, no arguments) ✔ codex.
- User template files with positional + named args under `/prompts:<name>` (codex 2025-08 → removed 2026-03-28, users pointed to `$skill-creator`) — codex chose **skills over templates** ([[skill-progressive-disclosure]]); plugin command Markdown auto-converted into skills on install (`2cd6ed7509`).
- Preconditions in the template text vs client-side checks: codex moved "AGENTS.md exists?" from a TUI filesystem check into the `/init` prompt so it is evaluated where tools run ([[client-side-check-wrong-host]]).
- Inline shell expansion in templates (opencode legacy; dropped in v2) → [[side-door-input-bypasses-hooks]].
- Run a command as a subtask in a subagent (opencode).
- Fold commands into skills (opencode v2 config).

## Implementations
- [[pi--prompt-template-expansion|pi]] — `~/.pi/agent/prompts`, `.pi/prompts`; expansion after input hooks and `/skill:`; repo dogfoods 6 maintainer templates.
- [[codex--prompt-template-expansion|codex]] — only built-in `/init` canned prompt today; user custom prompts (`/prompts:`) existed 2025-08 → 2026-03 and were replaced by skills.
- [[opencode--prompt-template-expansion|opencode]] — markdown commands with `$1..$N` / `$ARGUMENTS`, `` !`cmd` `` shell expansion (no permission check), `@file`; MCP prompts and skills as commands; `subtask` runs in a subagent; dropped from v2 config.

## Failures
- [[client-side-check-wrong-host]]
- [[generic-context-file-bloat]]

## Related
[[skill-progressive-disclosure]] · [[system-prompt-override]] · [[extension-event-hooks]] · [[harness-package-distribution]] · [[project-trust-gate]] · [[remote-host-trust]] · [[context-file-hierarchy]]
