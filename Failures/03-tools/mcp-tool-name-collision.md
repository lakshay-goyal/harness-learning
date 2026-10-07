---
type: failure
concepts: [mcp-integration, code-mode]
harnesses: [pi, opencode]
---
**Symptom** — Codemode scripts called the **wrong MCP tool** when two tools differed only by `-` vs `_` (`read-file` / `read_file`); pi tool names didn't match the codemode identifiers scripts used.

**Root cause** — MCP names kept `-` in the pi tool name but were mapped to `_` for JS identifiers — a non-injective mapping between two identifier spaces.

**Fix · [[pi]]** — `b29db895c` 2026-09-30 (#10239): MCP tool and namespace names replace every non-`[A-Za-z0-9_]` with `_` (like Codex) so the pi name **is** the codemode identifier; all colliding tools of a server get a sha256-8 hash suffix (order-independent); server names differing only in `-`/`_` are rejected; max 64 chars (`packages/coding-agent/src/extensions/mcp/tools.ts:48-93`).

**Lesson** — When tool names are mapped into another identifier space, make the mapping injective (one canonical sanitized name) and resolve collisions deterministically.

Related: [[mcp-integration]] · [[code-mode]] · [[tool-name-mapping-not-invertible]] · [[pi--mcp-integration|pi]]

**Fix · [[opencode]]** `b223a29603` 2025-08-11 names sanitized (`[^a-zA-Z0-9_-]` → `_`, `server_tool`, `packages/opencode/src/mcp/catalog.ts:117-119`) — still lossy, collisions unverified in practice. `a131811cdc` 2026-06-23 switched to `mcp__server__tool`, reverted the same day `947e0017f5` because users' permission rules referenced the legacy names. `6c12c32fb1` 2026-06-24 resource keys percent-encoded after `:` in client names collided. Code mode namespaces: "last write wins" (`cb93114424:packages/codemode/codemode.md:59-61`). Extra lesson: tool names are part of the user's config surface; renaming is a breaking change. See [[opencode--mcp-integration]].
