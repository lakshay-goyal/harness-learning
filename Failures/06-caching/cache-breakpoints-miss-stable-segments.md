---
type: failure
concepts: [cache-breakpoint-placement]
harnesses: [pi, opencode]
---
**Symptom** — Stable request segments were re-billed on every turn even though caching was on:
- Anthropic tool schemas missed cache whenever the transcript changed (#3260).
- User messages sent as plain strings never got a `cache_control` marker, so conversation history wasn't cached.
- OpenRouter (Anthropic-format markers via Chat Completions) never placed the conversation breakpoint on tool results, so tool-heavy turns didn't advance the cached prefix.

**Root cause** — Breakpoints only on system + last message: tools had no breakpoint of their own; marker code only handled array-shaped content; the "last message" walk skipped `tool` role messages.

**Fix · [[pi]]**
- `111a31e4d` 2026-02-02 "apply cache_control to string user messages" — string content upgraded to `[{type:"text", text, cache_control}]` (HEAD `packages/ai/src/api/anthropic-messages.ts:1499-1508`; same pattern in completions `openai-completions.ts:1160-1185`).
- `1c016cb01` 2026-04-16 (#3260) "cache Anthropic tools separately from transcript" — breakpoint on last tool definition "so tool schemas can be cached independently from transcript updates" (HEAD `anthropic-messages.ts:1204, 1608`, gated by `supportsCacheControlOnTools`).
- `bc41f612d` 2026-07-22 "cache OpenRouter tool results" — last-message walk includes `tool` messages (`openai-completions.ts:1110-1121`).
- Follow-on for tool *changes*: see [[late-tool-change-rewrites-cache]] (`9e05370b2`, `b271b0a52`).

**Fix · [[opencode]]** Legacy marks only the first two system messages (+ last two non-system); a plugin `experimental.chat.system.transform` that pushed extra system entries shifted the stable segments out of the marked positions → `72ebaeb8f7` 2025-12-15 (#5550) "rejoin system prompt if experimental plugin hook triggers to preserve caching": everything after the header re-joined so there are exactly two system blocks (`packages/opencode/src/session/llm/request.ts:68-78`). Legacy still has no tool breakpoint; v2 `942630eb4a` 2026-05-10 auto policy marks last tool + last system + latest user message (`packages/llm/src/cache-policy.ts:18-22`).

**Lesson** — Give every independently-stable segment its own breakpoint and normalize content shapes before placing markers; check the marker lands on whatever message type actually ends the turn.

Related: [[cache-breakpoint-placement]] · [[cache-miss-accounting]] · [[pi--cache-breakpoint-placement|pi]]
