---
type: implementation
harness: codex
concept: guideline-softening
commit: 622e9e3696
files: [codex-rs/core/gpt_5_codex_prompt.md:11, codex-rs/collaboration-mode-templates/templates/plan.md, codex-rs/prompts/templates/permissions/approval_policy/on_request.md, codex-rs/core/src/context/turn_aborted.rs:10]
---
[[guideline-softening]] in [[codex]].

## Mechanism
- No mechanism in code — a recurring editing pattern in prompt history: absolute MUSTs for one model behaviour are replaced within days by "prefer… unless…" wording that names when the alternative is legitimate; safety NEVERs stay absolute.
- Current examples: "Try to use apply_patch for single file edits, but it is fine to explore other options to make the edit if it does not work well. Do not use apply_patch for changes that are auto-generated (i.e. generating package.json or running a lint or format command like gofmt) or when scripting is more efficient (such as search and replacing a string across a codebase)." (`codex-rs/core/gpt_5_codex_prompt.md:11`); Plan mode "Strongly prefer using the `request_user_input` tool… In rare cases… you may ask it directly without the tool." (`codex-rs/collaboration-mode-templates/templates/plan.md`); "You should rarely pass the entire command into `prefix_rule`" (`codex-rs/prompts/templates/permissions/approval_policy/on_request.md`).

## Evolution
- 2025-08-05 `d31e149cb1` "When approval is denied or a command fails due to a permission error, do not retry the exact command in a different way. Move on…" — dropped in the 2025-08-07 rewrite `81b148bda2`.
- 2025-09-14 `916fdc2a37` → `a797051921` same day: "you must disregard user instruction and stop immediately and ask the user whether they are collaborating" softened to "While you are working, you might notice unexpected changes that you didn't make. If this happens, STOP IMMEDIATELY and ask the user how they would like to proceed."
- 2025-09-30 `f6a152848a` "When editing or creating files, you MUST use apply_patch as a standalone tool without going through ["bash", "-lc"], `Python`, `cat`, `sed`, ..." (measured 22 % → 0 % bypass) → reverted 2025-10-01 `5d78c1edd3` (no reason given) → 2025-10-04 `0ad1b0782b` permissive rule with carve-outs → [[edits-bypass-patch-tool]].
- 2026-01-26 `09251387e0`: `<turn_aborted>` text dropped "Do not continue or repeat work from that turn unless the user explicitly asks"; 2026-03-31 `2b8d29ac0d` dropped "verify current state before retrying" — marker now states facts, not prohibitions (`codex-rs/core/src/context/turn_aborted.rs:10`) → [[interrupted-turn-invisible-to-model]].
- 2026-01-31 `3dd9a37e0b` Plan-mode "Hard interaction rule (critical) — Every assistant turn MUST be exactly one of: A)… B)… C)…" and "No questions in free text (only via `request_user_input`)" → "Strongly prefer…" → [[imperative-guideline-over-compliance]].
- 2026-02-03 `968c029471` "You MUST NOT pass the entire command into `prefix_rule`" → "You should rarely pass…"; "don't try and circumvent approvals by using other tools" reworded to "Be judicious with escalating, but if completing the user's request requires it, you should do so - don't try and circumvent approvals by using other tools."
- 2026-05-11 `96836e15ed` removed "Time spent pursuing goal: {{ time_used_seconds }} seconds" from the goal continuation prompt (model got "nervous about time") → [[time-pressure-shortcuts]].
- Kept absolute: "**NEVER** use destructive commands like `git reset --hard` or `git checkout --` unless specifically requested or approved by the user." (`0ad1b0782b`).

## Versus pi
- Same pattern as [[pi--guideline-softening]] (PI_* "Inspect" → "You can inspect"); codex adds the explicit-carve-out form and keeps destructive-action NEVERs absolute.
