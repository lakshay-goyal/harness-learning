---
type: implementation
harness: pi
concept: cache-breakpoint-placement
commit: b30a6dd77
files: [packages/ai/src/api/anthropic-messages.ts:79-93, packages/ai/src/api/anthropic-messages.ts:1167-1230, packages/ai/src/api/anthropic-messages.ts:1483-1509, packages/ai/src/api/anthropic-messages.ts:1608, packages/ai/src/api/openai-completions.ts:1077-1191, packages/ai/src/api/bedrock-converse-stream.ts:870-892, packages/ai/src/api/bedrock-converse-stream.ts:917-919, packages/ai/src/api/bedrock-converse-stream.ts:1125-1131]
---
[[cache-breakpoint-placement]] in [[pi]].

## Mechanism
Placement is per adapter in pi-ai; the harness only chooses retention ([[cache-retention-control]]) and session id ([[session-affinity-cache-routing]]).

**Anthropic Messages** (`packages/ai/src/api/anthropic-messages.ts`) — `cache_control = {type:"ephemeral", ttl:"1h"?}`, none when retention `none` (`:79-93`):
1. **System blocks**: every system text block marked; OAuth sends 2 blocks (Claude Code identity + pi prompt), both marked (`:1167-1192`).
2. **Last tool definition** if `supportsCacheControlOnTools` (`:1204, 1608`) — "tool schemas can be cached independently from transcript updates" (`1c016cb01` #3260). With native tool changes the marker sits on the last *initial* tool, followed by `__pi_deferred_placeholder__` (`:1205-1219`) → [[cache-stable-prompt-prefix]].
3. **Last message** if role user/system: its last block when text/image/tool_result/tool_addition/tool_removal; string content upgraded to a text block so it can carry the marker (`:1483-1509`; `111a31e4d`).
- Managed-effort trailing system marker inserted *after* conversion (`insertThinkingLevelMessages`, `:1160, 1518`) so it is never the marked block.
- Mid-conversation system updates are held back and flushed before the next assistant message (`:1319-1328`) — the last-message marker then lands on the system update when it is last (`tool_addition` included in markable types).
- Max ~4 markers (OAuth: 2 system + tool + last message).

**OpenAI Chat Completions with `cacheControlFormat:"anthropic"`** (OpenRouter `anthropic/*` auto-detected, `openai-completions.ts:1642`): only if retention≠none; `ttl:"1h"` iff long && `supportsLongCacheRetention` (`:1077-1087`); `applyAnthropicCacheControl` → first system/developer message (last text part), last tool, last user/assistant/**tool** message with text walking backwards (`:1089-1135`; tool results count since `bc41f612d` "cache OpenRouter tool results"); string content → `[{type:"text", text, cache_control}]` (`:1160-1185`).

**Amazon Bedrock Converse** — `supportsPromptCaching` (`bedrock-converse-stream.ts:870-892`): requires "claude" in id/name (else only `AWS_BEDROCK_FORCE_CACHE=1`); Claude 5 (fable/opus/sonnet-5), any `-4-`, claude-3-7-sonnet, claude-3-5-haiku; Nova has automatic caching. Two `cachePoint`s: after system text (`:917-919`) and end of last user message (`:1125-1131`); long → `ttl: ONE_HOUR`; long retention not compat-gated on Bedrock.

**OpenAI Responses / Codex / Azure / Mistral** — no markers; implicit prefix caching keyed by `prompt_cache_key` (+ explicit mode on GPT-5.6+) → [[session-affinity-cache-routing]], [[cache-retention-control]].

**Google Gemini / Vertex** — ignores `cacheRetention`/`sessionId`; no explicit context caching (open question in findings 02; deliberate absence unverified).

## Constants
| name | value | path:line |
|---|---|---|
| Anthropic marker | `{type:"ephemeral"}` (+`ttl:"1h"` when long & supported) | `anthropic-messages.ts:88-92` |
| markable last-block types | text, image, tool_result, tool_addition, tool_removal | `anthropic-messages.ts:1491-1497` |
| Bedrock force flag | `AWS_BEDROCK_FORCE_CACHE=1` | `bedrock-converse-stream.ts:880` |
| Bedrock long TTL | `CacheTTL.ONE_HOUR` | `bedrock-converse-stream.ts:919, 1131` |

## Evolution
- `d442bbcc1` 2026-01-13: Bedrock prompt caching for Claude.
- `111a31e4d` 2026-02-02: `cache_control` applied to string user messages (were never cached).
- `1a4d153d7` 2026-03-14 (#2053): Bedrock caching limited to Claude (previously any model with cache pricing → API errors).
- `31d59f851` 2026-03-18 (#2346): `AWS_BEDROCK_FORCE_CACHE` for application inference profiles.
- `1c016cb01` 2026-04-16 (#3260): breakpoint on last tool — tool schemas cached separately from transcript.
- `4cd4cfd98` 2026-04-23: long retention compat flag.
- `5a07d946e` 2026-04-26 (#3527), `ed4bc7308` 2026-04-28: check/normalize `model.name` for opaque profile ARNs.
- `114bacf34` 2026-07-02 (#6235): Claude 5 IDs were missed by detection.
- `bc41f612d` 2026-07-22: OpenRouter breakpoint can land on tool results.
- `9e05370b2` 2026-09-16 / `b271b0a52` 2026-10-02: deferred placeholder + `tool_addition` blocks keep tool-list marker stable across tool changes.

## Evidence commits
`d442bbcc1`, `111a31e4d`, `1a4d153d7`, `31d59f851`, `1c016cb01`, `4cd4cfd98`, `5a07d946e`, `ed4bc7308`, `114bacf34`, `bc41f612d`, `9e05370b2`, `b271b0a52`.

## Quirks
- Runtime `detectCompat` vs generator compat duplicate the `~anthropic/` alias detection differently (`openai-completions.ts:1642` vs `generate-models.ts:749-750`) — which wins for custom models matching built-in URLs is (unverified).
- Last-message marker on Anthropic only when last role is user/system — after an assistant prefill there is no conversation marker.
- Capability detection by id substring is the recurring failure source (Bedrock ARNs, Claude 5 names).

## Failures
- [[cache-breakpoints-miss-stable-segments]]
- [[capability-sniffing-misses-opaque-ids]]
- [[late-tool-change-rewrites-cache]]
