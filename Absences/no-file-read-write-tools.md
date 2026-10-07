---
type: absence
harnesses: [codex]
---
# no-file-read-write-tools

No read_file / write_file / grep / ls / edit-by-string tools: the model reads and searches via the shell and changes files only through `apply_patch`.

**What's missing**
- Handler directory `codex-rs/core/src/tools/handlers/` has none; tool inventory `codex-rs/core/src/tools/spec_plan.rs:1082-1372`. Surface: `exec_command` / `write_stdin` + `apply_patch` + `update_plan` (opt-in) + `view_image` + optional web search / MCP / dynamic tools.
- Prompt: "Use `apply_patch` for manual code edits. Do not create or edit files with `cat` or other shell write tricks" (bundled model instructions, `codex-rs/models-manager/models.json:1408`).

**Evidence of decision**
- Experimental `list_dir` (`226215f36d` 2025-10-07), `grep_files` (`f52320be86` 2025-10-08), `read_file` indentation mode (`0026b12615` 2025-10-09) gated per model via `experimental_supported_tools` (`e0b38bd7a2` 2025-10-03).
- `read_file` dropped for gpt-5-codex `f3b4a26f32` 2025-10-05; handlers deleted `14c35a16a8` 2026-03-25 "chore: remove read_file handler" (611 deletions) and `178c3b15b4` 2026-03-25 "chore: remove grep_files handler"; `list_dir` deleted `70807730f5` 2026-05-05 because "nothing in the current model catalog advertises it via `experimental_supported_tools`".

**Implication**
- Read semantics are reconstructed for the UI by `parse_command` ([[shell-command-intent-parsing]]); patch format is a trained skill of the model family ([[patch-envelope-edit]]).
- Models are trained for this surface, so dedicated read tools added no value; contrast pi's [[minimal-default-toolset]] (read/bash/edit/write/grep/find/ls) — [[dedicated-vs-shell-tools]].

Related: [[file-read-tool]] · [[search-tools]] · [[search-replace-edit]] · [[minimal-default-toolset]] · [[dedicated-vs-shell-tools]] · [[no-binary-detection-in-read]] · [[Absences]]
