---
type: failure
concepts: [secret-handling, signed-reasoning-replay]
harnesses: [pi]
---
**Symptom** — OpenAI Responses (and later Azure OpenAI Responses) requests were sent without `store:false`, so every conversation — prompts, file contents, tool output, any credentials the agent printed — was retained server-side by the provider by default, although pi replays full context itself and never needs server-side state.

**Root cause** — Provider API default (`store` true) inherited silently; a stateless harness was opting into provider-side retention it didn't use.

**Fix · [[pi]]** — `d1fce2ba1` 2026-02-06 "disable OpenAI Responses storage by default (closes #1308)" (`packages/ai/src/api/openai-responses.ts:337` at HEAD: `store: false`); `84cdd0240` 2026-06-09 Azure (#5530, `packages/ai/src/api/azure-openai-responses.ts:203`; `packages/coding-agent/CHANGELOG.md:1662`). Stateless replay keeps reasoning via `include: ["reasoning.encrypted_content"]` ([[signed-reasoning-replay]]).

**Lesson** — Default provider data-retention flags to off when the harness replays context itself; every new provider adapter must be audited for retention defaults.

Related: [[secret-handling]] · [[signed-reasoning-replay]] · [[opaque-reasoning-payload-lost]] · [[pi--secret-handling|pi]]
