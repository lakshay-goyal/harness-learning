---
type: failure
concepts: [unified-provider-api]
harnesses: [pi]
---
**Symptom** Browser and Vite builds of pi-ai broke with `ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING` (#1814). Bundlers followed imports into Node-only code (AWS SDK, `node:http`, `node:crypto`).

**Root cause** `Function`-based dynamic imports, and a static module graph that reached Node-only adapters and OAuth flows.

**Fix · [[pi]]**
- `668ebc094` / `e0754fdbb` 2026-03-04: Bedrock is lazy-loaded, the OAuth runtime moved to the `/oauth` subpath, and `Function`-based imports were removed (#1814).
- `6442536b1` 2026-07-16: OAuth flows load through a variable import specifier with a `.ts`→`.js` rewrite, and Bun binaries register static loaders (`packages/ai/src/auth/oauth/load.ts:1-80`). The same trick is used for Bedrock (`packages/ai/src/api/bedrock-converse-stream.lazy.ts:4-24`).
- Guard comment: Node builtins use string-concatenated specifiers, "NEVER convert to top-level imports - breaks browser/Vite builds" (`packages/ai/src/env-api-keys.ts:1-24`).
- The root `index.ts` is side-effect free (`packages/ai/src/index.ts:4-8`).

**Fix · [[pi]]** (restricted runtimes beyond the browser) — AJV schema compilation blocked on Cloudflare Workers → `validateToolArguments()` falls back to no validation (0.61.0, #2395; `packages/ai/CHANGELOG.md:1415`, hash unverified); MCP `StreamableHttpTransport` failed every request on Workers with `Illegal invocation` → call `fetch` without a receiver (mcp 0.99.2, #10188; `packages/mcp/CHANGELOG.md:46`, hash unverified). Empty `process.env` in Bun sandboxes → see [[env-credential-discovery-misfires]].

**Lesson** Keep platform-specific adapters behind lazy, bundler-opaque imports so the core stays portable.

Related: [[unified-provider-api]] · [[async-init-race-caches-wrong-value]] · [[pi--unified-provider-api|pi]]
