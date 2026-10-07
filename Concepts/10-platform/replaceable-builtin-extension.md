---
type: concept
stage: architecture
tier: candidate
aliases: ["builtin:<name>", "-builtin:<name>", "replaceable: true", "minimal-core-extension-first", "pi's core is minimal", "built-in extensions"]
harnesses: [pi]
---
Minimal-core rule: the core only gains general mechanisms; in-box features (MCP, code mode, tool search, local-model provider) are implemented on the public plugin API, load by default, and can be disabled or displaced by name.

## Why
- Keeps the core small and reviewable ("PRs that bloat the core will likely be rejected") while still shipping batteries.
- Proves the plugin API is sufficient: if the built-in feature needs private hooks, third parties can't compete.
- Avoids double features: a third-party MCP plugin and the built-in one would both connect the same servers — replaceable built-ins step aside instead of conflicting.
- Lets a harness reverse a "will never ship X" stance (pi MCP: absent → built-in) without changing the core contract ([[no-builtin-mcp-reversed]]).

## Design space
- **Feature in core** (most harnesses) vs **feature as built-in plugin** (pi since 2026-09-29) vs **feature only as third-party package** (pi pre-2026-09 for MCP/sub-agents/plan mode).
- **Disable mechanism**: settings flag per feature (`--no-mcp`) vs uniform resource toggle (`-builtin:<name>`, `--no-extensions`) — pi has both.
- **Conflict policy** with same-named third-party registration: error (pi default between ordinary plugins) vs replaceable built-in silently drops out with a warning (pi).
- **Embedding**: built-ins auto-loaded in SDK too vs only in CLI (pi: SDK must add them manually).
- **Escape hatch for absent features**: README pointers + example plugins (pi: sub-agents, plan mode, permission gate, todo, sandbox examples).

## Implementations
- [[pi--replaceable-builtin-extension|pi]] — `builtInExtensions` list (`llama.cpp`, `codemode`, `tool-search`, `mcp`), resolved as `builtin:<name>` resources; replaceable ones omitted when another extension registers the same tool/command/flag.

## Failures
- (none mined specific to the mechanism)

## Related
[[extension-event-hooks]] · [[plugin-tools]] · [[runtime-plugin-loading]] · [[harness-package-distribution]] · [[mcp-integration]] · [[code-mode]] · [[deferred-tool-loading]] · [[minimal-default-toolset]] · [[no-subagents-core]] · [[no-plan-mode]] · [[no-permission-prompts]] · [[no-todo-tool]] · [[no-builtin-mcp-reversed]]
