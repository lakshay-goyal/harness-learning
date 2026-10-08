---
type: implementation
harness: pi
concept: cache-stable-prompt-prefix
commit: b30a6dd77
files: [packages/coding-agent/src/core/system-prompt.ts:155-191, packages/ai/src/api/anthropic-messages.ts:197-209, packages/ai/src/api/anthropic-messages.ts:1204-1230, packages/coding-agent/src/extensions/tool-search/tool.ts:215-219, packages/coding-agent/src/extensions/mcp/index.ts:1142-1148, packages/ai/src/utils/transcript.ts:122-142, packages/durable/src/harness/context.ts:170-180, packages/durable/docs/spec.md:3206-3208]
---
[[cache-stable-prompt-prefix]] in [[pi]].

## Mechanism
- **No volatile facts in the system prompt**: no date/time (removed `f4e9ca746`; `git grep 'Current date'` empty in coding-agent/agent src at HEAD), no model name, no session id; those go to env vars ([[env-vars-as-context]]). cwd normalized to "/" (`system-prompt.ts:183`) and docs paths absolute per install (`config.ts:444-456`) → constant per session.
- **Sectioned, patch-only prompt**: changes become appended section patches instead of a rewritten head (`diffSystemPromptSections`, `system-prompt.ts:217-229`) → [[transcript-carried-system-prompt]].
- **Byte-stable tool declarations**:
  - `tool_search` description "does not list the searchable tools or their namespaces, so it stays the same while tools are registered, for example when MCP servers connect" (`extensions/tool-search/tool.ts:215-219`).
  - codemode description no longer includes deferred tools, tool counts, or MCP server instructions "so it no longer changes when MCP servers connect" — volatile server list moved to the `mcp_servers` **section** (`CHANGELOG.md:197` #10212; `extensions/mcp/index.ts:1142-1148`); "Pi appends the new section to the conversation instead of changing tool declarations, so earlier messages stay cached" (`docs/mcp.md:202`).
  - Declaration equality via normalized JSON (`toToolDeclaration` strips TypeBox symbol keys/undefined, same key order; `transcript.ts:122-142`) so re-registration of an identical tool is a no-op, not a redeclare.
- **Preseeded placeholder** (Anthropic native tool changes): `__pi_deferred_placeholder__` (`defer_loading:true`, "Reserved placeholder. Never available. Never call this.") declared from the first request — "Anthropic adds hidden prompt scaffolding for mid-conversation tool changes; declaring this placeholder from the first request keeps that scaffolding in the cached prefix, so the first tool change does not invalidate the cache (measured: full miss without it)" (`anthropic-messages.ts:197-209`); request-level tool list then never changes (`:1205-1219`). Requires ≥1 initial tool (Anthropic rejects all-deferred lists, `:1136-1142`).
- **Leading system message**: durable `leadWithSystem` moves a system message preceded only by user messages to the front — "Providers treat only a leading system message as the initial prompt and tool set; without it, a later tool change rewrites the request's tool list and invalidates the whole prompt cache" (`packages/durable/src/harness/context.ts:170-180`; `92216fa15`).
- **Determinism contract** (durable spec): "Renderers must be deterministic for equal inputs: any change in rendered text, such as an embedded timestamp, appends a system delta and invalidates provider prompt caches." (`packages/durable/docs/spec.md:3206-3208`); footguns "Unstable prompt text" and "Moving default selections … each change appends `pi.system` entries and misses the provider prompt cache" (`spec.md:4715-4720`).
- Replay-side prefix stability: virtual-model routers should return `previous` for continuation "keeps prompt caches and thinking signatures valid. Switching models between turns is allowed but loses the prompt cache." (`docs/virtual-models.md:84`); Z.AI `clear_thinking:false` so replayed `reasoning_content` stays in the cached prefix (`b91bdd5a3` #6083).
- Hidden-declarations projection filters the whole transcript with the *current* hidden set "so the projected declarations stay consistent across requests and only change when the loadout does" (`agent-session.ts:1760-1784`).

## Constants
| name | value | path:line |
|---|---|---|
| placeholder tool | `__pi_deferred_placeholder__`, `defer_loading:true` | `anthropic-messages.ts:204-209` |
| beta | `inline-tools-2026-09-15` | `anthropic-messages.ts:195` |
| MCP section caps | 4096 chars / 250 per server | `extensions/mcp/index.ts:158, 163` |

## Evolution
- `b1c2c32e2` 2025-11-12: `Current date and time: <locale string>` appended to system prompt.
- `2a0f23928` 2025-12-11: precedent in `mom` bot — "remove dynamic timestamp from system prompt for better cache hits".
- `4b9e6006f` 2026-03-13 (#2131): ISO date only, "so prompt prefixes stay cacheable across reloads and resumed sessions" (CHANGELOG `:3124`).
- `f81acc667` 2026-04-17 (+`7f55605aa`, #2814): date built from components — "deterministic across runtimes and locales" (CHANGELOG `:2454`).
- `f4e9ca746` 2026-07-14 (#6621, @davidbrai): date removed — "Fixed system prompt cache invalidation across dates" (CHANGELOG `:1252`) → [[no-date-in-prompt]].
- `b91bdd5a3` 2026-06-29 (#6083): Z.AI preserve thinking for cache-stable replay.
- `9e05370b2` 2026-09-16 (#9548): sections + deltas + placeholder.
- `028c0ec56`/`e029c3ed0` 2026-09-30 + CHANGELOG `:196-198` (#10212): codemode/tool_search descriptions stabilized; `mcp_servers` section.
- `b271b0a52` 2026-10-02: `inline-tools` beta; redefinitions no longer resend the tool list.
- `92216fa15` 2026-10-06 (#10542): durable `leadWithSystem` with live e2e gate `test/system-order-cache-e2e.test.ts`.

## Evidence commits
`b1c2c32e2`, `2a0f23928`, `4b9e6006f`, `f81acc667`, `7f55605aa`, `f4e9ca746`, `b91bdd5a3`, `9e05370b2`, `028c0ec56`, `e029c3ed0`, `b271b0a52`, `92216fa15`.

## Quirks
- Three-step date saga (time → date → locale-stable date → none) took 8 months; each step cited cache hits.
- Consequence of no date: model has no "today" unless it runs `date`; no tool/extension supplies it (unverified beyond grep).
- Forced system prompts (`before_agent_start` returning `systemPrompt`) are projected onto the head each request — stable only if the extension's text is stable ([[system-prompt-override]]).
- Prefix stability still broken by compaction (summary replaces history) and by model switches (counted as misses by [[cache-miss-accounting]]).

## Durable variant (packages/durable)
- Section renderers re-run every request; only diffs appended as positional `pi.system` entries; after a head cut a complete baseline is written and older retained deltas omitted (`spec.md:3223-3234`; `packages/durable/src/harness/prompt.ts:58-83`). `leadWithSystem` fixes storage-order ≠ request-order (`context.ts:170-180`).

## Failures
- [[volatile-system-prompt-prefix]]
- [[mcp-startup-blocks-and-description-churn]]
- [[late-tool-change-rewrites-cache]]
