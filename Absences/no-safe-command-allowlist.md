---
type: absence
harnesses: [codex]
---
# no-safe-command-allowlist

The built-in "known safe commands" allowlist was deleted; auto-run decisions rest on the OS sandbox, user/admin exec-policy rules and a tiny dangerous-command denylist.

**What's missing**
- `is_known_safe_command` (536 lines + 503-line Windows variant) removed.

**Evidence of decision**
- `942af8447b` 2026-08-19 "Retire the untrusted approval policy (#39630)": "Remove the known-safe command allowlist. Projects marked untrusted now request approval for every command unless an explicit exec policy rule allows it."
- The list needed constant flag-level patching: `6cf4b96f9d` ripgrep flags, `c9e2def494`, `a1641743a8`, and `6e10142199` safe commands bypassing deny-read.

**Implication**
- Curated read-only command lists are a maintenance and bypass liability; codex moved trust to declarative rules ([[command-rule-policy]]) + containment ([[os-level-sandbox]]) + heuristics ([[dangerous-command-heuristics]]).

Related: [[command-rule-policy]] · [[dangerous-command-heuristics]] · [[approval-policy-modes]] · [[os-level-sandbox]] · [[no-permission-prompts]] · [[Absences]]
