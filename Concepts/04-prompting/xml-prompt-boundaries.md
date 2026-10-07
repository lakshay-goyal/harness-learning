---
type: concept
stage: messages
tier: must-have
aliases: ["<cwd>", "<skills>", sections, "<env>", ContextualUserFragment::type_markers, matches_marked_text, ContentItemKind, "<environment_context>", "<permissions instructions>", "<user_shell_command>", "<turn_aborted>", "<model_switch>"]
harnesses: [pi, opencode, codex]
---
Fence each prompt section and each injected document with XML tags (optionally with attributes like `path`) instead of markdown headings, so harness structure can't collide with content structure.

## Why
- Injected AGENTS.md files use `#`/`##` headings themselves; heading-delimited prompts become ambiguous ([[markdown-boundaries-ingested-inconsistently]]).
- Named tags double as addresses for later patches ("Updated system prompt section "rules"") → [[transcript-carried-system-prompt]].
- Missing separators glue sections together (cwd concatenated with appended text).

## Design space
- Markdown headings (pi until May 2026; still used for split-turn summarizer after XML+jargon caused refusals).
- **XML tags per section + per-file tags with path attribute** (**pi chose**).
- Escaping injected content (pi escapes skill metadata only; context file content unescaped).
- Out-of-band channel for harness remarks (durable `<harness>` → [[harness-diagnostics-channel]]).
- **Two layers: Markdown headings for the (static) base prompt, XML fences for every harness-injected fragment** ✔ codex; XML few-shot tags in instructions replaced by Markdown lists (`968c029471`).
- **Markers as the harness's own classifier**: case-insensitive start+end marker match to recognize injected user-role messages vs real user input (rollback, compaction keep-set, UI) + a machine `ContentItemKind` tag carried as metadata ✔ codex.
- Real XML with escaping and attributes for structured state (`<environment_context>` with `&amp;`… escaping, `<permission_profile type="…">`, `<network enabled="true">`) ✔ codex.
- Hybrid: Markdown H1 title + XML body (`# AGENTS.md instructions for <dir>` + `<INSTRUCTIONS>`) ✔ codex.
- Mixed: XML for harness-generated blocks, plain-text headers for instruction files (opencode).

## Implementations
- [[pi--xml-prompt-boundaries|pi]] — every non-preamble section `<name>…</name>`; `<project_instructions path>`; `<available_skills>`; `<summary>`.
- [[codex--xml-prompt-boundaries|codex]] — ~60 fragment types with `type_markers()` + `ContentItemKind`; Markdown base prompt; XML-escaped `<environment_context>`.
- [[opencode--xml-prompt-boundaries|opencode]] — XML for harness blocks (`<env>`, `<available_skills>`, `<mcp_instructions>`, `<diagnostics>`, `<system-reminder>`); instruction files only get a plain `Instructions from: <path>` header.

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
- [[injected-summary-indistinguishable]] (05-context)

## Related
[[context-file-hierarchy]] · [[transcript-carried-system-prompt]] · [[harness-diagnostics-channel]] · [[split-turn-summary]] · [[transcript-serialization-for-summary]] · [[message-role-layering]]
