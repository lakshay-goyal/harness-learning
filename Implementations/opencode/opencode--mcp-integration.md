---
type: implementation
harness: opencode
concept: mcp-integration
commit: ecc4916b5a
files: [packages/opencode/src/mcp/index.ts:7-9, packages/opencode/src/mcp/index.ts:38, packages/opencode/src/mcp/index.ts:442-470, packages/opencode/src/mcp/index.ts:661-663, packages/opencode/src/mcp/catalog.ts:11-12, packages/opencode/src/mcp/catalog.ts:68-79, packages/opencode/src/mcp/catalog.ts:117-119, packages/opencode/src/mcp/oauth-provider.ts:11, packages/opencode/src/session/tools.ts:27-32, packages/opencode/src/session/tools.ts:136-388, packages/opencode/src/session/system.ts:121-137, packages/core/src/v1/config/mcp.ts:21]
---
[[mcp-integration]] in [[opencode]].

## Mechanism
### Legacy runtime
- Transports: local `StdioClientTransport`; remote StreamableHTTP then SSE fallback, configurable headers (`packages/opencode/src/mcp/index.ts:7-9`).
- OAuth with dynamic registration, loopback callback port 19876 (or configured), tokens in `<data>/mcp-auth.json` mode 0600 (`packages/opencode/src/mcp/oauth-provider.ts:11`; `packages/opencode/src/mcp/auth.ts:37,80`).
- Timeouts: connect/list `DEFAULT_TIMEOUT = 30_000` (`index.ts:38`; `catalog.ts:11`); per call = server `timeout` ?? `experimental.mcp_timeout` ?? SDK default, with `resetTimeoutOnProgress: true` (`index.ts:661-663`).
- Re-list on `ToolListChanged` notifications (`index.ts:442-470`); catalogs paginated up to `MAX_LIST_PAGES = 1_000` (`catalog.ts:12`).
- Names: `sanitize(server) + "_" + sanitize(tool)`, sanitize = `[^a-zA-Z0-9_-]` → `_` (`catalog.ts:117-119`) → [[mcp-tool-name-collision]].
- Exposure: declared directly, one permission ask per call keyed on the tool name with `always: ["*"]`; content flattened (text joined, images as data-URL attachments, resources inlined) and truncated like built-ins (2000 lines / 50 KB) (`packages/opencode/src/session/tools.ts:136-210`) → [[tool-output-truncation]].
- Resources: three generic tools `list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource`, added only if some server has resources, gated by `read` permission on `mcp:<server>:*` (`tools.ts:27-32,170-185`); blobs ≤ 10 MiB.
- MCP prompts become slash commands → [[prompt-template-expansion]].
- Server `instructions` go into the system prompt as `<mcp_instructions><server name>` unless all of that server's tools are denied (`packages/opencode/src/session/system.ts:121-137`) → [[xml-prompt-boundaries]].
- Code mode flag on → MCP tools not declared; reachable only through `execute` (`tools.ts:388`) → [[code-mode]].

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_TIMEOUT` | 30 000 ms | `packages/opencode/src/mcp/index.ts:38` |
| `MAX_LIST_PAGES` | 1 000 | `packages/opencode/src/mcp/catalog.ts:12` |
| `OAUTH_CALLBACK_PORT` | 19876 | `packages/opencode/src/mcp/oauth-provider.ts:11` |
| `MAX_MCP_RESOURCE_BLOB_BYTES` | 10 MiB | `packages/opencode/src/session/tools.ts:32` |

## Evolution
- 2025-07-07 `0d50c867ff` raw content arrays corrupted sessions; 2025-07-25 `e8eaa77bf1` official SDK transports (streamable HTTP hang).
- 2025-08-11 `b223a29603` tool names sanitized.
- 2025-10-23 `3c7b229d8b` `tool.execute.after` can modify MCP output → [[tool-result-rewriting]].
- 2025-12-30 `ed4ce67cdc` configurable call timeout; 2026-01-03 `586e7347bd` 5 s connect timeout; 2026-01-04 `e5abe1e78b` "bump default to 30 seconds (lots of people complained about 5…)"; 2026-01-15 `dd1f981d23` per-server; 2026-06-10 `f43b0d3afd` catalog requests; 2026-06-16 `a98d5732c0` progress resets → [[mcp-startup-blocks-and-description-churn]].
- 2026-01-13 `80e1173ef7` close old client before reassignment; 2026-03-26 `2e6ac8ff49` close transport on failed connects; 2026-03-01 `c4c0b23bff` kill orphaned MCP child processes → [[mcp-child-processes-orphaned]].
- 2026-06-07 `b5cb9aae7f` respect server capabilities; 2026-06-08 `0efc334ff9` paginate; 2026-07-05 `2b34df94fa` metadata across pages.
- 2026-06-13 `c7dee9c609` recover expired sessions; 2026-07-30 `c1ee3c6e34` stop SSE reconnect loops.
- 2026-06-14 `dfb616f067` `isError` handled → [[mcp-error-result-treated-as-success]].
- 2026-06-23 `a131811cdc` `mcp__server__tool` names, reverted same day `947e0017f5` (existing permission configs referenced legacy names); 2026-06-24 `6c12c32fb1` resource key collisions.
- 2026-06-23 `3f3f120825` resource read tools; 2026-06-24 `e8e83afbce` server instructions in context; 2026-07-27 `921b1c6a34` SDK v2.

## Quirks / drift
- Schema doc still says "Defaults to 5000 (5 seconds) if not specified" (`packages/core/src/v1/config/mcp.ts:21`) while code uses 30 000.
- Sanitization is lossy (`a-b`, `a.b`, `a b` all → `a_b`); collisions unverified in practice.

Contrast: pi exposes MCP script-only through codemode by default and connects in the background → [[pi--mcp-integration|pi]].
