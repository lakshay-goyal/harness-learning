---
type: failure
concepts: [signed-reasoning-replay]
harnesses: [pi]
---
**Symptom** — After the system prompt, tools or effort changed mid-session, Claude returned a persistent 400 `Invalid signature in thinking block`. This happened both on direct Anthropic and on Bedrock.

**Root cause** — Anthropic binds signed thinking to the exact prefix it was produced under: system prompt, tools and effort. Mid-conversation system and tool updates (9e05370b2) changed that prefix.

**Fix · [[pi]]**
- `4e69b0c28` (2026-09-02), Anthropic managed-effort models: always send `thinking:{type:"adaptive", block_binding:{prefix_mismatch_behavior:"drop_block"}}`. The stated purpose is "so prefix mismatches can be dropped instead of surfacing as persistent 400 responses" (`packages/ai/src/api/anthropic-messages.ts:1233-1241`).
- The same commit adds per-turn effort markers: `insertThinkingLevelMessages` replays `{role:"system", output_config:{effort}}` before each historical assistant turn (`:1518-1532`). It also adds the `anthropic_input_transformations` diagnostic (`:871-883`).
- `69f0be6f0` (2026-10-02), #10324: the same `drop_block` plus the `thinking-binding-controls-2026-08-01` beta on Bedrock, only for Opus 4.7+, Sonnet 5 and Fable 5 (`packages/ai/src/api/bedrock-converse-stream.ts:792-806,1265-1279`). Opus and Sonnet 4.6 reject the field with "block_binding: Extra inputs are not permitted". GovCloud omits it (`:1244-1252`).

**Lesson** — Changing the system prompt or tools mid-session invalidates replayed reasoning. Ask the server to drop stale reasoning instead of failing the session, and gate that capability per model generation.

Related: [[signed-reasoning-replay]] · [[thinking-level-abstraction]] · [[transcript-carried-system-prompt]] · [[pi--signed-reasoning-replay|pi]]
