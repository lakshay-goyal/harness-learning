---
type: concept
stage: messages
tier: candidate
aliases: [prompt templates, "$ARGUMENTS", "$1", ".pi/prompts", slash commands, argument-hint]
harnesses: [pi]
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

## Implementations
- [[pi--prompt-template-expansion|pi]] — `~/.pi/agent/prompts`, `.pi/prompts`; expansion after input hooks and `/skill:`; repo dogfoods 6 maintainer templates.

## Failures
- (none specific recorded)

## Related
[[skill-progressive-disclosure]] · [[system-prompt-override]] · [[extension-event-hooks]] · [[harness-package-distribution]] · [[project-trust-gate]]
