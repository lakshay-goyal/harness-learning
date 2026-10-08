---
type: implementation
harness: opencode
concept: system-prompt-override
commit: ecc4916b5a
files: [packages/opencode/src/session/llm/request.ts:56-78, packages/opencode/src/session/llm/request.ts:99, packages/plugin/src/index.ts:291, origin/v2:packages/core/src/plugin/optimize.ts:81]
---
[[system-prompt-override]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Replace**: an agent's `prompt` (config agent or markdown agent) replaces the family prompt entirely; env, instructions, MCP and skills sections are still appended (`packages/opencode/src/session/llm/request.ts:60-62`) → [[agent-profiles]].
- **Append per message**: a user message's `system` field is joined at the end (`request.ts:62`).
- **Append via config**: `instructions` files/URLs → [[context-file-hierarchy]].
- **Plugin mutation**: `experimental.chat.system.transform` receives the `system` array and may mutate it arbitrarily (`request.ts:68-72`; hook type `packages/plugin/src/index.ts:291`). If it pushed entries and left the first one intact, everything after the head is re-joined so at most 2 system blocks reach the provider (`request.ts:73-77`) → [[cache-breakpoint-placement]].
- OpenAI OAuth (ChatGPT plan) sends the system as `options.instructions` instead of system messages (`request.ts:99`).
### v2 runtime / origin/v2
- Agent `system` set with `item.system ??=` so users can override the built-in one-liner (`packages/core/src/plugin/agent.ts:120-123`).
- Family plugins skip agents with their own `system` (`origin/v2:packages/core/src/plugin/optimize.ts:81`).
- Spec flags the legacy hook as unresolved: it "can mutate the assembled baseline system prompt arbitrarily", incompatible with an immutable cache baseline (`CONTEXT.md:225`).

## Constants
| name | value | path:line |
|---|---|---|
| max system blocks after plugin transform | 2 | `packages/opencode/src/session/llm/request.ts:73-77` |

## Evolution
- 2025-12-15 `72ebaeb8f7` "rejoin system prompt if experimental plugin hook triggers to preserve caching" → [[volatile-system-prompt-prefix]].
- 2026-01-25 `94dd0a8dbe` identity spoof moved from core into the auth plugin's system transform → [[provider-identity-shim]].

## Quirks / drift
- No SYSTEM.md / APPEND_SYSTEM.md file convention; overriding the base prompt requires defining an agent.

Contrast: pi offers append vs replace files/flags and structured section options chained across handlers → [[pi--system-prompt-override|pi]].
