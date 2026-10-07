---
type: implementation
harness: opencode
concept: context-file-hierarchy
commit: ecc4916b5a
files: [packages/opencode/src/session/instruction.ts:60-68, packages/opencode/src/session/instruction.ts:74, packages/opencode/src/session/instruction.ts:95-107, packages/opencode/src/session/instruction.ts:114-169, packages/opencode/src/session/instruction.ts:179-221, packages/opencode/src/tool/read.ts:355-356, packages/opencode/src/session/prompt.ts:691, packages/opencode/src/session/prompt.ts:1262-1271, packages/core/src/instruction-context.ts:35-74, packages/opencode/src/command/template/initialize.txt]
---
[[context-file-hierarchy]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Global**: first existing of `~/.config/opencode/AGENTS.md`, `~/.claude/CLAUDE.md` (latter unless `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT`) — one file only (`packages/opencode/src/session/instruction.ts:60-62,114-120`).
- **Project**: candidate names `AGENTS.md`, `CLAUDE.md`, `CONTEXT.md` (deprecated); for the **first name with any match**, `findUp` from cwd to worktree root adds every ancestor copy — "The first project-level match wins so we don't stack AGENTS.md/CLAUDE.md from every ancestor" (`instruction.ts:65-67,122-133`). Disabled by `OPENCODE_DISABLE_PROJECT_CONFIG`.
- **Config `instructions`**: globs (relative = glob-up cwd → worktree; `~/`; absolute) and http(s) URLs fetched with a 5 s timeout, concurrency 4; failures silently become empty (`instruction.ts:95-103,135-167`).
- Rendering: each file `Instructions from: <abs path>\n<content>` inside the single system message (`instruction.ts:166-167`), after the env block and before MCP instructions and skills (`packages/opencode/src/session/prompt.ts:1262-1271`). Re-read from disk every step.
- **Nested, on read**: `Instruction.resolve` walks from the read file's dir up to (not including) the instance root; files not in system paths, not loaded by an earlier read (`metadata.loaded`) and not claimed for this assistant message are appended to the read output inside `<system-reminder>` (`instruction.ts:179-221`; `packages/opencode/src/tool/read.ts:355-356`). Claims cleared at the end of each step (`prompt.ts:691`) → [[ephemeral-reminder-injection]].
- `/init` generates AGENTS.md: "Every line should answer: 'Would an agent likely miss this without help?' If not, leave it out" (`packages/opencode/src/command/template/initialize.txt`, `897d83c589`) → [[generic-context-file-bloat]].
### v2 runtime
- `InstructionContext`: global `<config>/AGENTS.md` + upward `AGENTS.md` from the Location dir to the project root, canonicalized and contained; no CLAUDE.md/CONTEXT.md yet (`packages/core/src/instruction-context.ts:40-74`).
- Changes delivered as transcript updates: "These instructions replace all previously loaded ambient instructions." / "Previously loaded instructions no longer apply." (`instruction-context.ts:35-37`) → [[transcript-carried-system-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| remote instruction timeout | 5 000 ms | `packages/opencode/src/session/instruction.ts:97` |
| remote fetch concurrency | 4 | `packages/opencode/src/session/instruction.ts:163` |

## Evolution
- 2025-07-03 `701107cda4` prompt reference CLAUDE.md → AGENTS.md.
- 2026-01-26 `39a73d4894` nested AGENTS.md resolved as the agent explores.
- 2026-01-28 `558590712d` parallel reads double-loaded AGENTS.md; 2026-02-02 `16145af480` duplicate injection when reading instruction files → [[context-file-loaded-twice-in-worktrees]].
- 2026-04-01 `897d83c589` `/init` tightened (was "Make it about 150 lines long", `2730e0c9cd` 2025-12-24).
- 2026-04-29 `00bb9836a6` order Global, Project, Skills.

## Quirks / drift
- Date in the env block and per-step disk re-read mean the system message changes on day rollover or any AGENTS.md edit → [[volatile-system-prompt-prefix]].
- Project discovery never mixes AGENTS.md and CLAUDE.md, but the global slot does accept `~/.claude/CLAUDE.md`.

Contrast: pi loads every candidate per ancestor to filesystem root, root → cwd, fenced in XML → [[pi--context-file-hierarchy|pi]].
