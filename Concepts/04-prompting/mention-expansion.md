---
type: concept
stage: messages
tier: variant
aliases: ["@file", "@agent", createUserMessage, resolvePart, "Called the Read tool with the following input", MAX_MCP_RESOURCE_BLOB_BYTES]
harnesses: [opencode]
---
`@file`, `@agent` and MCP-resource mentions are resolved when the message is created. They are stored as synthetic parts.

## Why
- A bare path forces a model round trip to read what the user already pointed at.
- Resolving at creation time pins the content the user saw; resolving at every request would change history.
- An agent mention must become an instruction the model can act on (call the subagent tool).

## Design space
- **Run the read tool inline and store call narration + output as synthetic text parts** (opencode) vs attach raw file content vs leave the path.
- Ranges: line ranges and LSP symbol ranges passed to the read call (opencode).
- Media (images, PDF) kept as file parts; MCP resources read and inlined with a blob cap (opencode 10 MiB).
- Agent mention → synthetic "call the task tool with subagent: X" text, with a "guaranteed to exist" hint when the task permission would deny it (opencode).
- Resolution concurrency vs order stability (opencode: unbounded concurrency, order preserved since `e35a4131d0`).
- Template-level `@file` in commands → [[prompt-template-expansion]].

## Implementations
- [[opencode--mention-expansion|opencode]] — `createUserMessage`/`resolvePart` in `packages/opencode/src/session/prompt.ts` turn file/agent/resource parts into synthetic text and file parts.

## Failures
- (none recorded)

## Related
[[synthetic-tool-call-injection]] · [[prompt-template-expansion]] · [[file-read-tool]] · [[mcp-integration]] · [[task-owned-subagent]] · [[ephemeral-reminder-injection]]
