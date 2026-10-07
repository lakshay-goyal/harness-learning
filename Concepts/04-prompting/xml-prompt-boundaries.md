---
type: concept
stage: messages
tier: candidate
aliases: ["<cwd>", "<skills>", sections]
harnesses: [pi]
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

## Implementations
- [[pi--xml-prompt-boundaries|pi]] — every non-preamble section `<name>…</name>`; `<project_instructions path>`; `<available_skills>`; `<summary>`.

## Failures
- [[markdown-boundaries-ingested-inconsistently]]

## Related
[[context-file-hierarchy]] · [[transcript-carried-system-prompt]] · [[harness-diagnostics-channel]] · [[split-turn-summary]] · [[transcript-serialization-for-summary]]
