---
type: implementation
harness: opencode
concept: xml-prompt-boundaries
commit: ecc4916b5a
files: [packages/opencode/src/session/system.ts:69-105, packages/opencode/src/session/system.ts:121-137, packages/opencode/src/skill/index.ts:321-338, packages/opencode/src/session/instruction.ts:166-167, packages/opencode/src/tool/skill.ts:48-60, packages/opencode/src/lsp/diagnostic.ts:27, packages/opencode/src/tool/read.ts:277-282]
---
[[xml-prompt-boundaries]] in [[opencode]]. Mixed: XML for harness-generated blocks, plain-text headers for instruction files.

## Mechanism
### Legacy runtime
- `<env>` block (working directory, workspace root, git yes/no, platform, `Today's date`) preceded by plain lines "You are powered by the model named …" / "Here is some useful information about the environment you are running in:" (`packages/opencode/src/session/system.ts:76-84`).
- `<available_references><reference><name><path><description>` for project references (`system.ts:86-103`) → [[project-references]].
- `<mcp_instructions><server name="…">` with indented server text (`system.ts:121-137`) → [[mcp-integration]].
- `<available_skills><skill><name><description><location>` with HTML-escaped location only (`packages/opencode/src/skill/index.ts:321-338`); skill bodies returned in `<skill_content name>` + `<skill_files>` (`packages/opencode/src/tool/skill.ts:48-60`).
- Tool results: `<diagnostics file="…">` (`packages/opencode/src/lsp/diagnostic.ts:26`), `<entries>` for directory reads (`packages/opencode/src/tool/read.ts:277-282`), `<task_result>` for subagents, `<system-reminder>` for harness notes → [[ephemeral-reminder-injection]].
- Instruction files are **not** fenced: `Instructions from: <abs path>\n<content>` (`packages/opencode/src/session/instruction.ts:166-167`) → [[context-file-hierarchy]].
- No escaping of injected file or server content.
### origin/v2 branch
- Base prompt uses markdown headings (`# Harness`, `# Communication`) and the identity plugin adds `# Your Model` → [[harness-identity]].

## Constants
| name | value | path:line |
|---|---|---|
| — | — | — |

## Evolution
- 2026-03-11 `0f6bc8ae71` / `f96e2d4222` verbose XML skill catalog in the system prompt.
- 2026-06-24 `e8e83afbce` `<mcp_instructions>`.

## Quirks / drift
- AGENTS.md content whose own markdown headings resemble prompt structure is unfenced (same risk as [[markdown-boundaries-ingested-inconsistently]]; no opencode incident found).

Contrast: pi fences each section and each context file with XML tags carrying a `path` attribute → [[pi--xml-prompt-boundaries|pi]].
