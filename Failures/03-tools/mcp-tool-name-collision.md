---
type: failure
concepts: [mcp-integration, code-mode]
harnesses: [pi, opencode, codex]
---
**Symptom** — Codemode scripts called the **wrong MCP tool** when two tools differed only by `-` vs `_` (`read-file` / `read_file`); pi tool names didn't match the codemode identifiers scripts used.

**Root cause** — MCP names kept `-` in the pi tool name but were mapped to `_` for JS identifiers — a non-injective mapping between two identifier spaces.

**Fix · [[pi]]** — `b29db895c` 2026-09-30 (#10239): MCP tool and namespace names replace every non-`[A-Za-z0-9_]` with `_` (like Codex) so the pi name **is** the codemode identifier; all colliding tools of a server get a sha256-8 hash suffix (order-independent); server names differing only in `-`/`_` are rejected; max 64 chars (`packages/coding-agent/src/extensions/mcp/tools.ts:48-93`).

**Fix · [[codex]]**
- Symptoms: sanitized names (`[A-Za-z0-9_]`) can collide; names were capped at 64 bytes though the Responses API accepts 128; normalized code-mode identifiers collided across dynamic / namespaced tools (`c126f206da` 2026-07-30); duplicate effective names from external tools / code mode / tool search reached the model; hyphenated server names broke listing (`a3b3e7a6cc` 2026-04-03).
- Deterministic 12-hex SHA-1 suffix of the raw identity (`server\0namespace\0connector\0callable\0raw`) on collision or length overflow, ≤128 bytes (`codex-rs/codex-mcp/src/tools.rs:120-316`); `1bfabb21fe` 2026-08-20 limit 64 → 128 bytes.
- `1e489adad0` 2026-08-05 opt-in `error_on_tool_collisions` fails the turn before sampling (`codex-rs/core/src/tools/spec_plan.rs:443-450`); any non-search tool named `tool_search` removed as a collision (`spec_plan.rs:392-427`).
- `0170860ef2` 2025-10-19 `mcp__` prefix introduced to make tool origin clear to the model; per-server opt-out `74e9d7efc4`.

**Lesson** — When tool names are mapped into another identifier space, make the mapping injective after every normalisation step (one canonical sanitized name, deterministic hash of the raw identity) and check uniqueness before sampling.

Related: [[mcp-integration]] · [[code-mode]] · [[tool-name-mapping-not-invertible]] · [[pi--mcp-integration|pi]] · [[codex--mcp-integration|codex]] · [[minimal-default-toolset]]
**Fix · [[opencode]]** `b223a29603` 2025-08-11 names sanitized (`[^a-zA-Z0-9_-]` → `_`, `server_tool`, `packages/opencode/src/mcp/catalog.ts:117-119`) — still lossy, collisions unverified in practice. `a131811cdc` 2026-06-23 switched to `mcp__server__tool`, reverted the same day `947e0017f5` because users' permission rules referenced the legacy names. `6c12c32fb1` 2026-06-24 resource keys percent-encoded after `:` in client names collided. Code mode namespaces: "last write wins" (`cb93114424:packages/codemode/codemode.md:59-61`). Extra lesson: tool names are part of the user's config surface; renaming is a breaking change. See [[opencode--mcp-integration]].
