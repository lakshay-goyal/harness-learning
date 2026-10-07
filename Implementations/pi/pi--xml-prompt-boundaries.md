---
type: implementation
harness: pi
concept: xml-prompt-boundaries
commit: b30a6dd77
files: [packages/coding-agent/src/core/system-prompt.ts:51-58, packages/coding-agent/src/core/system-prompt.ts:79-86, packages/coding-agent/src/core/system-prompt.ts:188-191, packages/ai/src/utils/text.ts:23-40, packages/coding-agent/src/core/skills.ts:358-388, packages/coding-agent/src/core/messages.ts:11-25, packages/durable/src/harness/tool.ts:483-485]
---
[[xml-prompt-boundaries]] in [[pi]].

## Mechanism
- Every system-prompt section except `preamble` is wrapped `<name>\n…\n</name>` (`packages/coding-agent/src/core/system-prompt.ts:188-191`); comment: "every other section is wrapped in a tag of the same name so the model can match later updates to it" (`:51-55`). Tag names constrained `/^[a-z][a-z0-9_-]*$/` (`:58`).
- Mid-conversation patches reference the same names: `Updated system prompt section "<name>":\n\n<text>` / `Removed system prompt section "<name>".` — "framed by name so the model can relate them to the leading prompt" (`packages/ai/src/utils/text.ts:23-40`) → [[transcript-carried-system-prompt]].
- Injected user content fenced with attributes: `<project_context>` → `<project_instructions path="…">…</project_instructions>` per file (`system-prompt.ts:79-86`) → [[context-file-hierarchy]]; skills `<available_skills><skill><name/><description/><location/></skill>` with `escapeXml` (`skills.ts:358-388`; `&,<,>,"` escaped); explicit `/skill:` invocation `<skill name="…" location="…">…</skill>` (`agent-session.ts:2159`).
- Other tag fences: compaction/branch summaries re-enter as `<summary>…</summary>` with preambles (`messages.ts:11-25`); summarizer input `<conversation>`, `<previous-summary>`, file lists `<read-files>`/`<modified-files>` (`compaction.ts:717-733`, `compaction/utils.ts:77-87`) → [[transcript-serialization-for-summary]], [[file-op-tracking]]; codemode output `<console_output>` + `==> text N/M <==` separators (`eb326d265` 2026-10-06, #10555); durable tool diagnostics `<harness>…</harness>` (`packages/durable/src/harness/tool.ts:483-485`) → [[harness-diagnostics-channel]].
- Counter-case: split-turn summary prompt moved **from** `<conversation>` XML **to** `# Conversation` / `# Instructions` markdown headings after Claude Fable refusals (`d192bd6dc` #9908/#9652) — tags + "PREFIX/SUFFIX" framing read as a reconstruct-hidden-content task → [[split-turn-summary]]. So pi uses XML for *fencing injected data inside instructions*, headings for *separating a data blob from the task*.
- CLI `@file` arguments become tag-fenced user text: text files → `<file name="/abs/path">\n<content>\n</file>`, images → image block plus `<file name="…"></file>` (or the resize hint / failure message inside the tag); empty files skipped, missing file = exit 1 (`packages/coding-agent/src/cli/file-processor.ts:30-87`). Initial prompt = piped stdin + file text + first CLI message concatenated in that order (`cli/initial-message.ts:20-42`). Images from `@file` are **not** resized at parse time — "AgentSession resizes these after extension hooks select the request model" (`main.ts:221-223`) → [[pi--image-normalization|image-normalization]].

## Constants
| name | value | path:line |
|---|---|---|
| section tag regex | `/^[a-z][a-z0-9_-]*$/` | `system-prompt.ts:58` |
| section tags (default order) | tools, rules, docs, addendum, project_context, skills, cwd, + extension (mcp_servers…) | `system-prompt.ts:152-186` |

## Evolution
- `b1c2c32e2` 2025-11-12: context files as `# Project Context` + `## <path>` markdown headings in system prompt.
- `05b7b8133` 2025-12-19: skills list → XML per Agent Skills standard.
- `e2fd651eb` 2026-05-16 (merge `8e6913711`, #4541, external @herrnel): "Updated system-prompt.ts to use xml boundaries during system and context file merging rather than using `##` so that agents are less likely to ingest a prompt with inconsistent boundaries." (custom-prompt path).
- `7577d3b8d` 2026-05-18 (merge `aad8cf660`, #4709): same for default prompt ("…less likely to ingest a prompt with unclear boundaries").
- `88619669e` 2026-05-06 (#4234): HTML export strips skill wrapper XML from user messages (UI side of fencing).
- `3dd4623ee` 2026-08-10 (#7887): missing newline glued cwd to appended custom content — boundary bug pre-tags.
- `9e05370b2` 2026-09-16 (#9548): "Available tools:"/"Guidelines:"/"Current working directory:" headers → `<tools>`/`<rules>`/`<cwd>` tags; tags double as patch addresses.
- `d192bd6dc` 2026-09-22: split-turn prompt de-XML'd (refusal fix).
- `1247476e6` 2026-09-16: evals' plain-text markers (`\nGuidelines:\n`, `Current working directory:`) broke on restructure → updated to XML sections.

## Evidence commits
`b1c2c32e2`, `05b7b8133`, `e2fd651eb`, `8e6913711`, `7577d3b8d`, `aad8cf660`, `88619669e`, `3dd4623ee`, `9e05370b2`, `d192bd6dc`, `1247476e6`, `eb326d265`.

## Quirks
- Context file *content* is not escaped — an AGENTS.md containing `</project_instructions>` can break the fence (no escaping in `renderProjectContext`, `system-prompt.ts:79-86`; consistent with [[no-prompt-injection-defense]]). Skills metadata *is* escaped.
- Branch summaries get double framing: stored summary starts with "The user explored a different conversation branch before returning here." and render adds "The following is a summary of a branch…<summary>" (`branch-summarization.ts:253-256`, `messages.ts:19-24`).

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
