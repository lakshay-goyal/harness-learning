---
type: tradeoff
concepts: [cache-stable-prompt-prefix, transcript-carried-system-prompt, cache-breakpoint-placement, cache-warming, session-affinity-cache-routing]
harnesses: [pi, opencode]
---
# prompt-cache-strategy

**Axis**: how does the harness keep the provider's prompt-cache prefix stable when the system prompt, tools or environment change?

| dimension | pi | opencode legacy | opencode v2 | evidence |
|---|---|---|---|---|
| Volatile facts in the system prompt | none: date removed (`f4e9ca746`), session details in env vars | `Today's date` + instruction files re-read every step in one system message | date is a context source; change appended as "Today's date is now: …" | pi → [[no-date-in-prompt]]; legacy `packages/opencode/src/session/system.ts:69-105`; v2 `packages/core/src/system-context/builtins.ts:31-39` |
| How prompt changes reach the model | sectioned prompt in the transcript; later changes are appended section patches | system rebuilt each request (collapsed to ≤ 2 messages) | **Context Epoch**: baseline stored durably and reused verbatim; one Mid-Conversation System Message per change at a safe turn boundary; new baseline only after compaction | pi `packages/coding-agent/src/core/system-prompt.ts:217-229` → [[transcript-carried-system-prompt]]; legacy `packages/opencode/src/session/llm/request.ts:74-78`; v2 `packages/core/src/session/context-epoch.ts:59-70`, `1af8dafd3e` |
| Tool-block stability | byte-stable declarations; mid-conversation tool add/remove; placeholder declared from the first request | tools sorted by name; task/code-mode descriptions rebuilt per step | registry projection per turn; max-step turn drops tools | pi `packages/ai/src/utils/transcript.ts:122-142`; legacy `packages/opencode/src/tool/registry.ts:265-289` |
| Breakpoints | per-provider adapters | first 2 system + last 2 messages (`ephemeral`) | `cache: "auto"`: last tool, last system part, latest user message; cap 4 | legacy `packages/opencode/src/provider/transform.ts:358-359`; v2 `packages/llm/src/cache-policy.ts:18-37` → [[cache-breakpoint-placement]] |
| Mid-run reminders | out-of-band messages deferred to turn end | plan/build reminders on the newest user message; queued-message wrapper removed for busting cache (`f092bafe88`) | — | → [[ephemeral-reminder-injection]], [[mid-run-user-input]] |
| Keep-alive | expected-value cache warming (1-token probe, ≥ $0.05 savings) | none | none | pi `packages/coding-agent/src/core/cache-warmer.ts:20-31` → [[cache-warming]] |
| Affinity | Codex WebSocket reuse | `promptCacheKey = sessionID`, `x-session-affinity` headers | same; `ses_` stripped for long ids (`747b8daafc`) | → [[session-affinity-cache-routing]] |

**Convergence**: opencode v2's Context Epoch is the same idea as pi's transcript-carried prompt, reached independently from the opposite starting point (legacy rebuilt the prompt every request). Both now append state changes instead of rewriting the head.

**When each wins**
- **Rebuild per request (opencode legacy)**: simplest; always fresh AGENTS.md and date. Fine for short sessions or providers without caching. Loses the cache on every day rollover or instruction edit.
- **Append deltas (pi, opencode v2)**: long sessions on Anthropic/OpenAI where cache reads are ~0.1× price. Costs: a transcript that carries system state, per-provider lowering (Gemini cannot take mid-conversation system messages, so it collapses), and a durable baseline to manage.
- **Warming (pi only)**: worth it only with an explicit cost model; idle users mostly don't return within the TTL.

Related: [[cache-stable-prompt-prefix]] · [[compaction-design]] · [[single-vs-per-model-system-prompt]] · [[volatile-system-prompt-prefix]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
