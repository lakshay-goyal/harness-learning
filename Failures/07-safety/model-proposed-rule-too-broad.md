---
type: failure
concepts: [command-rule-policy, sandbox-escalation-retry, permission-state-prompt]
harnesses: [codex]
---
**Symptom** — Once the model could propose a reusable "always allow" `prefix_rule` with an escalation request, it suggested prefixes that would auto-approve arbitrary code (`["bash","-lc"]`, `["python3"]`, `["python","-"]`, `["git"]`), whole commands (too specific to reuse), or rules for heredoc commands — which users rubber-stamp.

**Root cause** — The rule format is an escalation channel: a user approving "this prefix forever" approves everything an interpreter prefix can run.

**Fix · [[codex]]** — Prompt: 2026-01-28 `996e09ca24` introduced `prefix_rule` with good/bad examples; 2026-02-03 `968c029471` "### Banned prefix_rules"; 2026-02-04 `8f17b37d06` "do not request [\"python3\"], [\"python\", \"-\"]… NEVER provide a prefix_rule if your command uses a heredoc or herestring."; 2026-03-20 `7754dd1b89` "…that would allow arbitrary scripting" + rules not evaluated for redirection/substitution/env vars/globs (`codex-rs/prompts/templates/permissions/approval_policy/on_request.md:44-57`). Code: 2026-02-12 `e6e4c5fa3a` "Restrict model-suggested rules (#11671)" — 88 `BANNED_PREFIX_SUGGESTIONS` (`codex-rs/core/src/exec_policy.rs:57-146`), rejected if an existing rule already matched, accepted only if it would actually allow every parsed sub-command (`codex-rs/core/src/exec_policy.rs:958-1021`).

**Lesson** — When the model drafts persistent permissions for a human to approve, validate them in code against a ban-list of code-execution prefixes and against the concrete command; the prompt alone is not the boundary.

Related: [[command-rule-policy]] · [[sandbox-escalation-retry]] · [[permission-state-prompt]] · [[codex--command-rule-policy|codex]] · [[codex--permission-state-prompt|codex]]
