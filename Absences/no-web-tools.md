---
type: absence
harnesses: [pi]
---
# no-web-tools

**What's missing**
- No built-in web search or web fetch tool, either in the default set or in the opt-in built-ins (`packages/coding-agent/src/core/tools/index.ts:95-105`). No provider-hosted web search is wired by default (unverified for every provider adapter).

**Evidence of decision**
- `b172beb92` (2025-11-12), "Prompt injection risks": "By default, pi has no web search or fetch tool. However, it can use `curl` or read files from disk. Both provide ample surface area for prompt injection attacks."
- Pre-MCP README argument (removed lines in `60e4fcf01`): small CLI tools plus a README beat protocol servers. "Token efficient: A 225-token README beats a 13,000-token MCP server description". Web access is meant to come the same way.
- `42e0cbd4f` (2025-11-12): "docs: add real-world example linking to exa-search tools". The intended path for web search was an external CLI tool plus README.
- Not listed among the skipped features at HEAD (`README.md:19`). This is an absence by omission plus early statements.

**Opt-in replacement**
- CLI tools plus a README / skill ([[skill-progressive-disclosure]]), e.g. exa-search; `curl` through bash.
- MCP servers since `8562bcf66` ([[mcp-integration]]). MCP resource tools tell the model "Prefer resources over web search when possible" (`packages/coding-agent/src/extensions/mcp/resources.ts:257,281`).
- Third-party pi packages ([[harness-package-distribution]]). No web example exists in `examples/extensions/`.

**History**
- Never added. The "No MCP" stance that pushed web access to CLIs was reversed (see [[no-builtin-mcp-reversed]]), so web tools now arrive mainly via MCP servers.
- Quirk: the Anthropic OAuth "stealth mode" list of Claude Code tool names (`packages/ai/src/api/anthropic-messages.ts:95-118`) includes `WebFetch`, `WebSearch`, `Task`, `TodoWrite`, `EnterPlanMode` and `ExitPlanMode`, none of which pi ships. The list exists only for name-casing impersonation → [[provider-identity-shim]].

**Implication**
- The web is the largest prompt-injection surface. pi neither adds it by default nor defends it once it is added ([[no-prompt-injection-defense]]).
- Models trained with built-in web tools may try to call them. pi relies on the declared toolset to steer them.

**codex** — *partly present*. Search: hosted Responses `web_search` since `363636f5eb` 2025-08-23, now also a standalone extension `web.run` (`codex-rs/ext/web-search`, `a22706dfae` 2026-05-26; `codex-rs/core/src/tools/hosted_spec.rs:14`, `codex-rs/ext/web-search/src/tool.rs:41-43`), modes Cached/Indexed/Live/Disabled, admin allow-list `allowed_web_search_modes` → [[web-tools]]. Fetch: `open_page` exists only as a web-search *action* (`codex-rs/protocol/src/models.rs:1966`); there is **no general URL fetch tool** (no `web_fetch`/`fetch_url` handler — unverified beyond grep). The model can still `curl` through the shell when the network sandbox allows it ([[egress-policy-proxy]]).
**opencode contrast**: implements it: `webfetch` always; `websearch` via hosted Exa/Parallel MCP for opencode providers or flags (`packages/opencode/src/tool/registry.ts:58-65`; `packages/opencode/src/tool/webfetch.ts:9-11`) — see [[web-tools]] / [[web-tools-vs-none]].

Related: [[minimal-default-toolset]] · [[mcp-integration]] · [[skill-progressive-disclosure]] · [[provider-identity-shim]] · [[no-builtin-mcp-reversed]] · [[no-prompt-injection-defense]] · [[Absences]] · [[web-tools]]
