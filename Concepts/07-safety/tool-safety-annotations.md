---
type: concept
stage: permissions
tier: candidate
aliases: [annotations, readOnlyHint, destructiveHint, idempotentHint, openWorldHint, ToolAnnotations]
harnesses: [pi]
---
Declarative, unverified per-tool hints (read-only / destructive / idempotent / open-world, MCP semantics) that policy plugins read to decide which calls need confirmation, instead of pattern-matching tool names or arguments.

## Why
- Name/regex-based gates (`rm -rf`, `sudo`) only know built-in tools; once MCP servers and plugins add arbitrary tools, a permission plugin has no way to tell a lookup from a delete.
- Lets the harness ship *metadata* without shipping *policy* (pi: "a permission extension can use them to decide which calls to confirm").

## Design space
- **None**: policy by tool name / argument regex (pi example `permission-gate.ts`, pre-0.99).
- **MCP-style boolean hints** carried on the tool definition and exposed to plugins — pi's choice since `8562bcf66` (0.99.0).
- **Defaults when hint missing**: MCP defaults (not read-only, maybe destructive, open-world) — pi follows; i.e. unknown ⇒ treated as risky by the doc's sample policy.
- **Trust in hints**: author-asserted, unverified (pi) vs. harness-verified/sandbox-enforced.
- **Harness-owned risk classes** (e.g. Codex approval categories) — pi docs show a snippet reproducing "the calls Codex asks approval for" from hints.
- **Built-in tools annotated** vs not — pi's built-ins carry no annotations at HEAD (only MCP tools and MCP resource tools do).

## Implementations
- [[pi--tool-safety-annotations|pi]] — `ToolAnnotations` on tool definitions; MCP tools copy boolean hints, MCP resource tools are `readOnlyHint:true`; exposed via `pi.getAllTools()`; consumed only by user policy code.

## Failures
- none recorded

## Related
[[tool-call-gate]] · [[mcp-integration]] · [[plugin-tools]] · [[deferred-tool-loading]] · [[no-permission-prompts]]
