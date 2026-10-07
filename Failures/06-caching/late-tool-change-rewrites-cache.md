---
type: failure
concepts: [transcript-carried-system-prompt, cache-stable-prompt-prefix, cache-breakpoint-placement]
harnesses: [pi]
---
**Symptom** — Adding, removing or redefining a tool mid-conversation (extension `setActiveTools`, MCP servers connecting, loadout changes) invalidated the whole Anthropic prompt cache: the next request was a full-price write. Variants:
- prompt/tool changes rewrote the leading system prompt + tool list (pre-#9548);
- the *first* native mid-conversation tool change was still a full miss (Anthropic adds hidden scaffolding);
- same-name redefinitions resent the whole tool list;
- in pi-durable runs, requests began with the user message because input was committed before the system prompt was rendered, so Anthropic treated the tool list as the head and every tool change rewrote it (#10542).

**Root cause** — Providers cache from the first byte; the tool list and system prompt sit at the top. Any change there — or a provider-side scaffolding block that appears on first use — rewrites the prefix. Storage order ≠ request order in the durable harness.

**Fix · [[pi]]**
- `9e05370b2` 2026-09-16 (#9548) "Mid conversation system messages": prompt sections + tool declarations live in the transcript; changes are appended deltas; Anthropic request `tools` fixed to initial tools + `__pi_deferred_placeholder__` declared from request 1 — "keeps that scaffolding in the cached prefix, so the first tool change does not invalidate the cache (measured: full miss without it)" (HEAD `packages/ai/src/api/anthropic-messages.ts:197-209, 1205-1219`).
- `b271b0a52` 2026-10-02 "define Anthropic mid-conversation tools inline": `inline-tools-2026-09-15` beta, later tools as `tool_addition{tool_definition}` / `tool_removal{tool_reference}`, so redefinitions no longer resend the tool list (`anthropic-messages.ts:195, 1339-1353`); `hasToolRedefinitions()` deprecated (`transcript.ts:179-197`).
- `92216fa15` 2026-10-06 (#10542, durable) `leadWithSystem` — "Move a system message that only user messages precede to the front … without it, a later tool change rewrites the request's tool list and invalidates the whole prompt cache" (`packages/durable/src/harness/context.ts:170-180`); live gate `test/system-order-cache-e2e.test.ts`.

**Lesson** — Keep tools/system at the very front and fixed; express changes as appended deltas, pre-declare anything that would otherwise change hidden scaffolding, and normalize provider-visible order independently of storage order.

Related: [[transcript-carried-system-prompt]] · [[cache-stable-prompt-prefix]] · [[cache-breakpoint-placement]] · [[deferred-tool-loading]] · [[stale-thinking-signature-after-prefix-change]] · [[pi--transcript-carried-system-prompt|pi]]
