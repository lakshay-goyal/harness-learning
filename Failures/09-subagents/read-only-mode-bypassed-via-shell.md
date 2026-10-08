---
type: failure
concepts: [plan-mode, shell-command-permission-parsing]
harnesses: [opencode]
---
**Symptom** — In plan mode the model modified files with the shell (`sed -i`, `tee`, `echo >`, `cat >`) because only the edit-class tools were blocked.

**Root cause** — The plan agent denies the `edit` permission but inherits `*: allow` for `bash`; "read-only" was enforced for the shell only by prompt text.

**Fix · [[opencode]]** — prompt escalation only:
- `a826936702` 2025-07-09 "you MUST NOT make any edits".
- `457386ad08` 2025-08-11 "fix plan mode bash tool making changes": "Bash tool must only run readonly commands".
- `ca3769b7fa` 2025-08-16: "CRITICAL: Plan mode ACTIVE … STRICTLY FORBIDDEN … Do NOT use sed, tee, echo, cat, or ANY other bash command to manipulate files … ZERO exceptions" (`packages/opencode/src/session/prompt/plan.txt:1-9`).
- Still no mechanical shell restriction at HEAD `ecc4916b5a` (`packages/opencode/src/agent/agent.ts:156-180`).

**Lesson** — A read-only mode must constrain the shell mechanically (deny `bash`, or allow only parsed read-only commands); capitalised prose is not enforcement.

Related: [[plan-mode]] · [[shell-command-permission-parsing]] · [[permission-ruleset]] · [[read-only-mode-bypass-via-subagent]] · [[opencode--plan-mode|opencode]]
