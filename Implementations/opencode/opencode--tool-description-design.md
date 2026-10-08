---
type: implementation
harness: opencode
concept: tool-description-design
commit: ecc4916b5a
files: [packages/opencode/src/tool/shell/prompt.ts:98-119, packages/opencode/src/tool/shell/prompt.ts:225-290, packages/opencode/src/tool/shell/shell.txt:7-21, packages/opencode/src/tool/edit.txt:4, packages/opencode/src/tool/write.txt:5, packages/opencode/src/tool/websearch.txt:11-14, packages/opencode/src/tool/skill.txt:5, packages/opencode/src/tool/registry.ts:277, packages/core/src/tool/bash.ts:109, packages/core/src/tool/apply-patch.ts:70-72, packages/opencode/src/tool/plan-exit.txt:1-13]
---
[[tool-description-design]] in [[opencode]].

## Mechanism
### Legacy runtime
- `.txt` description per tool; shell is templated: `${intro} ${os} ${shell} ${tmp} ${workdirSection} ${commandSection}` with per-shell variants (bash, pwsh, PowerShell 5.1, cmd) and limits interpolated from the truncation constants ("If the output exceeds ${maxLines} lines or ${maxBytes} bytes … written to a file") (`packages/opencode/src/tool/shell/prompt.ts:98-119`).
- Habit-replacing parameters plus a ban: `workdir` + "AVOID using `cd <directory> && <command>`" (`prompt.ts:112`); "Do NOT use `head`, `tail`…" because output is already spilled → [[shell-cd-chaining]], [[model-truncates-command-output]].
- Tool → shell-command mapping in the shell description ("Read files: Use Read (NOT cat/head/tail)", `prompt.ts:100-103`) → [[shell-cat-instead-of-read-tool]].
- Cache stability: per-project `${directory}` removed from the shell description twice; websearch date quantized to the year; skill tool description made static ("must match one of the skills listed in your system prompt", `packages/opencode/src/tool/skill.txt:5`) → [[cache-stable-prompt-prefix]].
- Dynamic part kept: task description appends the permitted subagent list (`packages/opencode/src/tool/registry.ts:277`).
- Plugin `tool.definition` hook can rewrite any description/schema.
- When/when-not lists: `plan_exit` (imported at `packages/opencode/src/tool/plan.ts:11`) gives three "Call this tool" and three "Do NOT call this tool" conditions, and states the side effect: it asks the user whether to switch to the build agent (`packages/opencode/src/tool/plan-exit.txt:1-13`) → [[plan-mode]].
### v2 runtime
- One-paragraph contracts stating authority and failure semantics: bash "with the host user's filesystem, process, and network authority" (`packages/core/src/tool/bash.ts:109`); apply_patch "Operations apply sequentially; if a later operation fails, earlier operations remain applied" (`packages/core/src/tool/apply-patch.ts:70-72`).
- origin/v2: tool usage lines rendered into the system prompt only for tools present → [[dynamic-tool-guidelines]].

## Constants
| name | value | path:line |
|---|---|---|
| shell.txt size | 6287 → 1269 bytes (`548648a3d9`) | `packages/opencode/src/tool/shell/shell.txt` |
| task.txt / todowrite.txt | 3931 → 2278 / 8845 → 2012 bytes (`548648a3d9`) | `packages/opencode/src/tool/` |

## Evolution (shell description = failure log)
- 2025-08-11 `22023fa9e7` opencode commit trailer instruction removed.
- 2025-11-19 `793542230f` "If and only if the user asks…"; 2025-12-04 `d469d7d441` "ONLY COMMIT IF THE USER ASKS YOU TO." → [[unrequested-git-commits]].
- 2025-12-06 `75a4dcbce8` `workdir`; 2025-12-11 `4e92f54415` "try to prevent the cd spam"; 2025-12-26 `160c8ab7cc` good/bad example.
- 2025-12-10 `de8460cb99` pasted Claude Code text incl. `run_in_background` → removed 2025-12-17 `751899eeec` → [[tool-description-drifts-from-implementation]].
- 2025-12-25 `281ce4c0c3` "DO NOT use it for file operations (reading, writing, editing, searching, finding files)".
- 2026-03-03 `e79d41c70e` no head/tail truncation.
- 2026-03-27 `15a8c22a26` `${directory}` removed for cache hits; reintroduced by pwsh support `b234370080`; removed again 2026-04-02 `38014fe448` → [[volatile-system-prompt-prefix]].
- 2026-04-24 `bb3509b5ff` amend only after verifying HEAD → later "do not amend the failed commit" (`shell.txt:18`) → [[amend-after-failed-commit]].
- 2026-04-30 `2283979199` pre-approved `${tmp}`; `3615d8e226` "already been created, already exists" → [[model-distrusts-preprovisioned-resource]].
- 2026-05-03 `3f459819ba` shell-aware prompts; 2026-05-16 `548648a3d9` Claude-Code-length git/PR protocol collapsed to 8 bullets.

## Quirks / drift
- Promises the mechanism does not keep: "persistent shell session" (`prompt.ts:259`); read-before-edit in `edit.txt:4`/`write.txt:5` (check deleted `76a141090e`); websearch "Domain filtering" (`websearch.txt:11`); max-steps "Tools are disabled" while legacy still sends tools → [[tool-description-drifts-from-implementation]].
- `profile()` still computes `gitCommands`, `createPrInstruction`, `createPrExample` that the template no longer references (`prompt.ts:225-287`, dead values).
- grep.txt sends counting to `rg` via bash, contradicting the shell mapping (`prompt.ts:100-103`).

Contrast: pi interpolates limits too but keeps descriptions short and moves rules into per-tool prompt snippets → [[pi--tool-description-design|pi]].
