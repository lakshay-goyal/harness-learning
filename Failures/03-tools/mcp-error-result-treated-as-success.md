---
type: failure
concepts: [mcp-integration, tool-error-as-result]
harnesses: [opencode]
---
**Symptom** — MCP tool results with `isError: true` were passed to the model as normal output; when both `content` and `structuredContent` existed, the model got only the JSON and lost the human-readable text.

**Root cause** — The adapter mapped MCP content without consulting the protocol's error flag, and preferred structured output.

**Fix · [[opencode]]**
- `dfb616f067` 2026-06-14 "handle tool result errors (#32244)".
- `fd213e6df6` 2026-06-29 "prefer content over structured output (#34505)": structured JSON used only when `content` is empty (`packages/opencode/src/mcp/catalog.ts:68-79`).

**Lesson** — Map protocol-level error flags into the harness's tool-error channel; give the model the text the server wrote for humans.

Related: [[mcp-integration]] · [[tool-error-as-result]] · [[structured-tool-output]] · [[signal-killed-command-reported-success]] · [[opencode--mcp-integration|opencode]]
