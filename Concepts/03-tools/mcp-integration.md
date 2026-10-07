---
type: concept
stage: tools
tier: candidate
aliases: [builtin:mcp, packages/mcp, mcp_servers section, "mcp__<server>__<tool>", mcp.json, mcp-lazy-connect, mcp-name-sanitization, cli-tools-over-mcp, "<server>_<tool>", list_mcp_resources, read_mcp_resource, mcp_timeout, mcp-auth.json, "<mcp_instructions>"]
harnesses: [pi, opencode]
---
How Model Context Protocol servers and their tools are surfaced to the model: configuration and trust, connection timing, name sanitization, exposure (declared, deferred, script-only), result conversion/truncation, auth, and list-change handling.

## Why
- MCP tool descriptions are large (pi's old stance: "A 225-token README beats a 13,000-token MCP server description") — declaring them all is expensive.
- Server names/tool names must map to provider- and script-safe identifiers injectively ([[mcp-tool-name-collision]]).
- Slow servers block the first prompt; tool lists that change as servers connect churn caches ([[mcp-startup-blocks-and-description-churn]]).
- Tools appear/disappear at runtime; loaded state must survive resume ([[deferred-tools-lost-on-resume]]); allowlists must not silently drop them ([[tool-allowlist-hides-mcp-tools]]).
- OAuth token rotation races across processes ([[oauth-refresh-token-rotation-lost]]).

## Design space
- **No MCP; CLIs + READMEs/skills instead** (pi 2025-11 → 2026-09; [[no-builtin-mcp-reversed]]).
- MCP as a replaceable built-in extension on general mechanisms (pi since 2026-09-29) vs core feature.
- Exposure default: declared (direct) vs searchable (deferred) vs **script-only (codemode, pi default)** vs hidden; per-tool override patterns.
- Connection: eager blocking vs **background connect, wait lazily only for what's needed** (pi).
- Server list in a patchable system-prompt section, not in tool descriptions (pi).
- Result handling: middle-truncation with spill (pi 20KB) vs head; structured content to scripts untruncated.
- Safety: project config gated by trust ([[project-trust-gate]]); annotations to policy plugins ([[tool-safety-annotations]]); every call through hooks ([[tool-call-gate]]); no retry of tool calls.
- Generic resource tools (list, list templates, read) gated by `read` permission (opencode).
- Server instructions in the system prompt, dropped when all of that server's tools are denied (opencode).
- Progress notifications reset the per-call timeout (opencode `resetTimeoutOnProgress`).

## Implementations
- [[pi--mcp-integration|pi]] — standalone `packages/mcp` client (stdio + streamable HTTP + OAuth), `mcp__server__tool` names ≤64 chars with hash suffix, default codemode exposure, background connect with 10 s first-prompt wait only for direct tools.
- [[opencode--mcp-integration|opencode]] — stdio / StreamableHTTP→SSE, OAuth loopback; tools declared directly as `server_tool` with a permission ask per call; 30 s connect; 3 generic resource tools; server instructions in a `<mcp_instructions>` system block; hidden behind `execute` in code mode.

## Failures
- [[tool-output-bypasses-truncation]]
- [[mcp-connect-timeout-too-short]]
- [[mcp-tool-name-collision]]
- [[oauth-refresh-token-rotation-lost]]
- [[mcp-startup-blocks-and-description-churn]]
- [[tool-allowlist-hides-mcp-tools]]
- [[deferred-tools-lost-on-resume]]
- [[mcp-child-processes-orphaned]]
- [[mcp-error-result-treated-as-success]]

## Tradeoffs
- [[mcp-builtin-vs-extension]]
- [[web-tools-vs-none]]

## Related
[[deferred-tool-loading]] · [[code-mode]] · [[minimal-default-toolset]] · [[replaceable-builtin-extension]] · [[project-trust-gate]] · [[tool-safety-annotations]] · [[subscription-oauth-auth]] · [[cache-stable-prompt-prefix]] · [[tool-call-gate]]
