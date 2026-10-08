---
type: implementation
harness: opencode
concept: mention-expansion
commit: ecc4916b5a
files: [packages/opencode/src/session/prompt.ts:65-72, packages/opencode/src/session/prompt.ts:699-790, packages/opencode/src/session/prompt.ts:795-870, packages/opencode/src/session/prompt.ts:909-960, packages/opencode/src/session/prompt.ts:974-990, packages/opencode/src/session/prompt.ts:995]
---
[[mention-expansion]] in [[opencode]].

## Mechanism
### Legacy runtime
- `createUserMessage` resolves every prompt part through `resolvePart`, `Effect.forEach(..., {concurrency: "unbounded"})` with order preserved (`packages/opencode/src/session/prompt.ts:699,995`).
- **File mention** (`@path`, `file:` URL): runs the `read` tool inline with `{filePath, offset, limit}`; line ranges come from the URL, or from an LSP `documentSymbol` range when a symbol is referenced (`prompt.ts:831-854`). Stored as synthetic text parts: `Called the Read tool with the following input: {…}` + the read output, or a failure line (`prompt.ts:795,861,936,955`) → [[synthetic-tool-call-injection]], [[file-read-tool]].
- Directories go through read as `application/x-directory` (`prompt.ts:811,909`); images/PDF stay file parts (`prompt.ts:68-71,1012`).
- **MCP resource** mention: "Reading MCP resource: …" then text inlined, supported binaries attached, others replaced by `[Binary MCP resource omitted: … exceeds …]`; blob cap 10 MiB (`prompt.ts:65,703-780`) → [[mcp-integration]].
- **Agent mention** (`@agent`): synthetic text "Use the above message and context to generate a prompt and call the task tool with subagent: <name>", plus " . Invoked by user; guaranteed to exist." when the caller's `task` permission would deny that agent (`prompt.ts:974-990`) → [[task-owned-subagent]].
- Resolution happens once at message creation; history keeps the resolved content, not the reference.
- Template-level `@file` in custom commands is expanded by the same path after template substitution → [[prompt-template-expansion]].

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_MCP_RESOURCE_BLOB_BYTES` | 10 MiB | `packages/opencode/src/session/prompt.ts:65` |
| attachable MIME | gif, jpeg, png, webp (+ PDF) | `packages/opencode/src/session/prompt.ts:66-71` |

## Evolution
- 2026-02-16 `e35a4131d0` "keep message part order stable when files resolve asynchronously".
- 2026-06-23 `3f3f120825` MCP resource read tools (resource mentions share the blob cap).

## Quirks / drift
- The narration says the Read tool was "called", but no tool call exists in the transcript; it is user-side text the model may imitate.

pi: no mention-expansion mechanism recorded in the atlas (unverified absence).
