---
type: implementation
harness: codex
concept: prompt-template-expansion
commit: 622e9e3696
files: [codex-rs/tui/assets/prompt_for_init_command.md:1, codex-rs/tui/src/chatwidget/slash_dispatch.rs:328, codex-rs/core-plugins/src/command_migration.rs:114]
---
[[prompt-template-expansion]] in [[codex]].

## Mechanism
- **`/init`** submits a fixed Markdown file as a normal user message (`codex-rs/tui/src/chatwidget/slash_dispatch.rs:328-330`): "Generate a file named AGENTS.md that serves as a contributor guide for this repository." Requirements: title "Repository Guidelines", "Keep the document concise. 200-400 words is optimal.", sections Project Structure / Build-Test-Dev Commands / Coding Style / Testing / Commit & PR ("Summarize commit message conventions found in the project's Git history"), optional "Agent-Specific Instructions" (`codex-rs/tui/assets/prompt_for_init_command.md:1-41`). No templating, no arguments.
- Precondition in the prompt: "Before writing, check whether AGENTS.md already exists in the current working directory. If it does, do not overwrite or modify it." (`codex-rs/tui/assets/prompt_for_init_command.md:2`) — the model checks in the execution environment, not the TUI host.
- **No user templates today**: custom prompts in `$CODEX_HOME/prompts` were removed (`48144a7fa4`); plugin `commands/` Markdown is converted into generated skills on plugin install (manifest `commands` field or `commands/` dir; skipped on unsupported templates, missing descriptions, name collisions, or generated skills > 4 KB) (`codex-rs/core-plugins/src/command_migration.rs:114-300`; `2cd6ed7509`) → [[skill-progressive-disclosure]].
- `/plan` accepts prompt args and pasted images (`3392c5af24`) → [[codex--plan-mode]].

## Constants
| name | value | path:line |
|---|---|---|
| /init AGENTS.md target length | 200-400 words | `codex-rs/tui/assets/prompt_for_init_command.md:10` |
| generated command-skill size cap | 4 KB | `2cd6ed7509` (commit body) |

## Evolution
- 2025-08-06 `ffe24991b7` "Initial implementation of /init (#1822)" — text unchanged since, except one sentence.
- 2025-08-28 `b8e8454b3f` "Custom /prompts (#2696)"; 2025-09-29 `80ccec6530` numeric args, `bf76258cdc` "Custom prompts begin with `/prompts:`"; 2025-09-30 `3592ecb23c` "Named args for custom prompts"; 2025-10-18 `c81e1477ae` prompt descriptions used; 2025-11-05 `fff576cf98` symlinked Markdown; 2025-11-24 `523dabc1ee` expansion with large pastes; 2026-01-20 `531748a080` "Prompt Expansion: Preserve Text Elements".
- 2026-03-18 `e5de13644d` startup deprecation warning for custom prompts, "with guidance to use `$skill-creator`"; 2026-03-28 `48144a7fa4` "Remove remaining custom prompt support (#16115)".
- 2026-06-09 `8e69d29521` "Reduce TUI legacy core dependencies (#26711)": added the AGENTS.md-exists sentence to `/init`; deleted the TUI `init_target.exists()` guard ("AGENTS.md already exists here. Skipping /init to avoid overwriting it.") because "checking the TUI's local filesystem for `/init` is incorrect" when the TUI drives a remote app-server; test `slash_init_skips_when_project_doc_exists` → `slash_init_does_not_depend_on_loaded_instruction_sources` → [[client-side-check-wrong-host]].
- 2026-07-15 `2cd6ed7509` "Migrate plugin commands into skills on install (#33411)".

## Versus pi
- [[pi--prompt-template-expansion]] keeps bash-style user templates (`$1`, `$ARGUMENTS`) alongside skills; codex had the same feature for 7 months and removed it, folding user macros into skills.
