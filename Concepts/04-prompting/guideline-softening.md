---
type: concept
stage: messages
tier: candidate
aliases: ["You can inspect…", "You must use this tool", over-compliance]
harnesses: [pi]
---
Downgrade imperative always-on rules to permissive or capability-describing wording once they cause over-compliance; keep imperatives only for trigger-scoped rules.

## Why
- An imperative in an always-present rule ("Inspect PI_* environment variables…") is executed every turn → wasted tool calls ([[imperative-guideline-over-compliance]]).
- Escalated wording ("You must…", ALL-CAPS `critical_rule`) fixes one model's under-compliance but persists after models change.

## Design space
- Imperative/MUST/NEVER rules (pi `42d7d9d9b`, Codex bridge).
- **Capability statement ("You can inspect…")** (**pi chose** `4e64de695`).
- **Named anti-pattern without intensity ("Use read … instead of cat or sed.")** (**pi**, `235b247f1`).
- Conditional scoping ("read only when the user asks about pi") (**pi**).
- Delete the rule entirely once models outgrow it (pi: "do NOT use cat or bash to display what you did").

## Implementations
- [[pi--guideline-softening|pi]] — escalate→de-escalate history for read and PI_* rules.

## Failures
- [[imperative-guideline-over-compliance]]
- [[shell-cat-instead-of-read-tool]]

## Related
[[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[env-vars-as-context]]
