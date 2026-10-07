---
type: concept
stage: compaction
tier: candidate
aliases: [CompactionDetails, "<read-files>", "<modified-files>", extractFileOpsFromMessage, computeFileLists, FileOperations]
harnesses: [pi]
---
Mechanically extract which files were read and modified from tool calls, carry the cumulative lists across compactions in entry metadata, and append them to the summary so the model still knows its working set.

## Why
- LLM summaries drop file names; after compaction the model re-explores or edits the wrong file. A deterministic list is cheap and exact.
- Lists must be cumulative: each compaction only sees new messages, so without carrying the previous lists files touched before the last compaction disappear.

## Design space
- **Extract from structured tool calls** (tool name ∈ {read, write, edit} with string `path`) ✔ pi — blind to `bash cat`/`sed -i`, grep, and plugin tools with other names (open gap).
- Include nested calls made from scripts (pi: `toolResult.nestedCalls` from codemode).
- Classification: modified = edited ∪ written; read-only = read − modified; sorted ✔ pi.
- Carry-over: entry `details` ✔ pi; only trust harness-generated details, not plugin-supplied ones (pi `!fromHook`).
- Rendering: XML tags appended after summary ✔ pi vs a section inside the LLM template vs separate system note.
- Branch summaries carry their own lists from nested branch summaries (not from compaction details).
- Dropped entirely in pi durable (no file-op tracking).
- **LLM-written mandatory section instead of mechanical lists**: `## Relevant Files` in the summary template, added after paths kept vanishing across compactions (opencode `78f85b1cd6`; not an implementation of this concept) → [[summary-template-drops-goals]].

## Implementations
- [[pi--file-op-tracking|pi]] — `FileOperations{read,written,edited}` from tool calls + nested calls; `CompactionDetails{readFiles,modifiedFiles}` carried if not extension-generated; appended as `<read-files>`/`<modified-files>`.

## Failures
- (none mined; gap: bash-driven file edits invisible — unverified impact)

## Related
[[auto-compaction]] · [[structured-compaction-summary]] · [[iterative-summary-update]] · [[branch-summary]] · [[code-mode]] · [[nested-tool-calls]]
