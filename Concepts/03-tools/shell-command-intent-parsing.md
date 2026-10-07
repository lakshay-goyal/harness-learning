---
type: concept
stage: tools
tier: candidate
aliases: [parse_command, ParsedCommand, "ParsedCommand::{Read, ListFiles, Search, Unknown}", parsed_cmd, codex-shell-command, command summarization]
harnesses: [codex]
---
When the model reads, lists and searches through a generic shell tool (no dedicated read/grep/ls tools), parse each command after the fact into semantic intents — read file X, list dir Y, search for Z, unknown — for UI summaries, approval prompts and telemetry.

## Why
- A shell-only tool surface ([[dedicated-vs-shell-tools]]) loses the "what is the agent doing" signal that dedicated tools give for free; raw `bash -lc "rg -n foo src | head"` is unreadable in a transcript view.
- Approval prompts and exploration summaries ("Read foo.rs", "Searched for bar") need the intent, not the argv.
- Mis-classifying a mutating command (e.g. `sed -i`, `rg --replace`) as a read would hide a write.

## Design space
- **Dedicated read/search/list tools, intent explicit in the tool name** (pi `read`/`grep`/`find`/`ls`, [[search-tools]], [[file-read-tool]]).
- **Shell only + post-hoc parser** (codex `parse_command`) — best effort, lossy ("~infinite expressiveness of an arbitrary command").
- Parser scope: unwrap `bash -lc` / PowerShell wrappers, pipelines, `cd &&` chains, `xargs`; flag in-place mutators as non-read (codex).
- Consumers: UI only vs also approvals and telemetry (codex: exec begin/end events, approval requests, user `!` commands, telemetry tagging).
- Safety classification is a separate parser ([[dangerous-command-heuristics]], [[approval-key-canonicalization]]) vs shared.

## Implementations
- [[codex--shell-command-intent-parsing|codex]] — `codex-rs/shell-command` `parse_command(argv) -> Vec<ParsedCommand>`; `parsed_cmd` attached to `ExecCommandBegin/End` events and approval requests.

## Failures
- (none recorded)

## Related
[[dedicated-vs-shell-tools]] · [[shell-execution]] · [[search-tools]] · [[file-read-tool]] · [[minimal-default-toolset]] · [[dangerous-command-heuristics]] · [[approval-key-canonicalization]] · [[terminal-scrollback-tui]]
