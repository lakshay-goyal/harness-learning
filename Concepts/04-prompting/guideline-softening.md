---
type: concept
stage: messages
tier: must-have
aliases: ["You can inspect…", "You must use this tool", over-compliance, "Strongly prefer…", "Try to use apply_patch"]
harnesses: [pi, opencode, codex]
---
Downgrade imperative always-on rules to permissive or capability-describing wording once they cause over-compliance; keep imperatives only for trigger-scoped rules.

## Why
- An imperative in an always-present rule ("Inspect PI_* environment variables…") is executed every turn → wasted tool calls ([[imperative-guideline-over-compliance]]).
- Escalated wording ("You must…", ALL-CAPS `critical_rule`) fixes one model's under-compliance but persists after models change.
- An absolute MUST can hit 100 % compliance in eval and still be reverted because it blocks legitimate alternatives (codex apply_patch MUST: 22 % → 0 % bypass, reverted next day, [[edits-bypass-patch-tool]]).

## Design space
- Imperative/MUST/NEVER rules (pi `42d7d9d9b`, Codex bridge; codex apply_patch MUST `f6a152848a`, Plan-mode "Hard interaction rule (critical)", "You MUST NOT pass the entire command into `prefix_rule`").
- **Capability statement ("You can inspect…")** (**pi chose** `4e64de695`).
- **Named anti-pattern without intensity ("Use read … instead of cat or sed.")** (**pi**, `235b247f1`).
- **Preference with explicit carve-outs naming when the alternative is legitimate** ("Try to use apply_patch for single file edits, but it is fine to explore other options… Do not use apply_patch for changes that are auto-generated… or when scripting is more efficient") ✔ codex (`0ad1b0782b`).
- **"Strongly prefer X… In rare cases… you may Y"** ✔ codex (Plan mode `3dd9a37e0b`); "You should rarely pass…" ✔ codex (`968c029471`).
- Conditional scoping ("read only when the user asks about pi") (**pi**).
- Delete the rule entirely once models outgrow it (pi: "do NOT use cat or bash to display what you did"; codex: truncation limits removed from prompt `570eb5fe78`).
- Counter-trend: escalation for *safety* rules stays absolute (codex "**NEVER** use destructive commands like `git reset --hard`…", `0ad1b0782b`).
- Escalation as the opposite move when prose is the only enforcement (opencode plan reminder: "STRICTLY FORBIDDEN … ZERO exceptions").
- Replace "when needed" with an enumerated list of valid cases (opencode codex question policy).

## Implementations
- [[pi--guideline-softening|pi]] — escalate→de-escalate history for read and PI_* rules.
- [[codex--guideline-softening|codex]] — apply_patch MUST reverted within a day; Plan-mode turn-shape rule and `prefix_rule` MUST softened within days; destructive-git NEVERs kept.
- [[opencode--guideline-softening|opencode]] — brevity, safety boilerplate, GPT autonomy and Codex question policy softened per family prompt; plan-mode reminder escalated instead; "Always use TodoWrite" dropped and re-added in 3 days.

## Failures
- [[model-echoes-work-via-shell]]
- [[imperative-guideline-over-compliance]]
- [[shell-cat-instead-of-read-tool]]
- [[edits-bypass-patch-tool]]
- Cross-group: [[interrupted-turn-invisible-to-model]] (01-loop) · [[time-pressure-shortcuts]] (01-loop)
- [[model-echoes-work-via-shell]] (04-prompting) — When summarizing what it had done, the model ran cat/heredoc/echo through bash to "display" its summary…
- [[autonomy-prompt-overreach]]
- [[over-commenting-code]]
- [[excessive-permission-questions]]

## Related
[[dynamic-tool-guidelines]] · [[minimal-system-prompt]] · [[env-vars-as-context]] · [[per-model-system-prompt]] · [[plan-mode]]
