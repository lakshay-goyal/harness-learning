---
type: concept
stage: tools
tier: candidate
aliases: [builtin:mcp, packages/mcp, mcp_servers section, "mcp__<server>__<tool>", mcp.json, mcp-lazy-connect, mcp-name-sanitization, cli-tools-over-mcp, codex-mcp, rmcp-client, codex_apps, DEFAULT_OPTIONAL_MCP_STARTUP_GRACE, tool_timeout_sec]
harnesses: [pi, codex]
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
- **MCP as core from the start** (✔ codex since 2025-05, own `codex-mcp` + `rmcp-client` crates) vs replaceable built-in extension (✔ pi).
- Connection: optional servers get a shared 1 s grace then are omitted from that turn; required / mentioned servers block; cached catalogs exposed before startup with read-only hints cleared (✔ codex).
- Exposure default **deferred** behind `tool_search` when the model supports it, else direct (✔ codex) vs codemode-only (✔ pi).
- Names: `mcp__<server>__<tool>`, sanitized to `[A-Za-z0-9_]`, 12-hex SHA-1 suffix on collision, ≤128 bytes, per-server prefix opt-out (✔ codex) vs ≤64 chars + sha256-8 (✔ pi).
- Approvals derived from annotations with pessimistic defaults; "Allow / Allow for this session / Allow and don't ask me again / Cancel" (✔ codex) → [[tool-safety-annotations]].
- Concurrency from `readOnlyHint` (✔ codex) → [[parallel-tool-execution]].
- `structuredContent` replaces content blocks in the model-visible output (✔ codex) → [[structured-tool-output]].
- Request `_meta` carries sandbox state + thread/turn ids for cooperating servers (✔ codex `codex/sandbox-state-meta`).
- Elicitations routed to user, LLM reviewer, or auto-declined by policy (✔ codex) → [[llm-approval-reviewer]].
- Bound persisted copies (rollout, history, events) separately from the model copy (✔ codex 1 MiB events, 64 KiB history preview).

## Implementations
- [[pi--mcp-integration|pi]] — standalone `packages/mcp` client (stdio + streamable HTTP + OAuth), `mcp__server__tool` names ≤64 chars with hash suffix, default codemode exposure, background connect with 10 s first-prompt wait only for direct tools.
- [[codex--mcp-integration|codex]] — core MCP: 30 s startup / 300 s call timeouts, 1 s optional grace, 32-entry 30 min catalog cache, deferred exposure, hashed 128-byte names, annotation-driven approvals and parallelism, elicitation routing.

## Failures
- [[mcp-tool-name-collision]]
- [[oauth-refresh-token-rotation-lost]]
- [[mcp-startup-blocks-and-description-churn]]
- [[tool-allowlist-hides-mcp-tools]]
- [[deferred-tools-lost-on-resume]]
- [[unannotated-mcp-tools-serialized]]
- [[unbounded-tool-output-overflows-context]]
- (07) [[mcp-annotation-defaults-unsafe]]
- (06) [[nondeterministic-tool-order-breaks-cache]]
- (01) [[side-phase-input-lost]]
- [[strict-tool-schema-rejections]] (02-model-interface) — Provider 400s on tool declarations:
- [[invented-schema-constraint]] (03-tools) — Untyped nodes in MCP / dynamic tool schemas were filled in as string by the sanitizer, inventing "a scalar…
- [[oauth-issuer-mixup-accepted]] (07-safety) — pi's MCP OAuth client exchanged an authorization code from a response naming a different issuer than the…

## Related
[[deferred-tool-loading]] · [[code-mode]] · [[minimal-default-toolset]] · [[replaceable-builtin-extension]] · [[project-trust-gate]] · [[tool-safety-annotations]] · [[subscription-oauth-auth]] · [[cache-stable-prompt-prefix]] · [[tool-call-gate]] · [[llm-approval-reviewer]] · [[elicitation-pause]] · [[tool-schema-normalization]] · [[parallel-tool-execution]]
