---
type: failure
concepts: [mcp-integration]
harnesses: [opencode]
---
**Symptom** Slow-starting stdio MCP servers, such as `npx` or `uvx` cold starts, failed to connect within the default timeout. "lots of people complained about 5…"

**Fix · [[opencode]]** `e5abe1e78b` 2026-01-04 "tweak: bump default to 30 seconds": `DEFAULT_TIMEOUT` went from `5000` to `30_000` (`packages/opencode/src/mcp/index.ts:38`, applied through `withTimeout(client.connect(t), timeout)` at `:226`).

**Lesson** Connect timeouts for subprocess servers must cover package-manager cold start. Measure them on first-run machines, not warm caches.

Related: [[mcp-integration]] · [[opencode--mcp-integration|opencode]] · [[mcp-builtin-vs-extension]] · [[Constants]]
