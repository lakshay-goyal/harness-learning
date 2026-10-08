---
type: failure
concepts: [cross-provider-handoff, signed-reasoning-replay, usage-cost-accounting]
harnesses: [pi, opencode]
---
**Symptom** — On Anthropic-compatible relays that report a different response model, and on server-side fallbacks, the next turn's signed thinking was treated as cross-model. It was converted to text or lost its signature, which broke reasoning continuity.

**Root cause** — The adapter overwrote `output.model` with the model id the server reported. `isSameModel` in transform-messages compares `assistantMsg.model === model.id` (`packages/ai/src/api/transform-messages.ts:95-98`), so a relabeled reply looked like it came from a foreign model.

**Fix · [[pi]]**
- `1283afd0d` (2026-09-17), #9188: `model` keeps the requested id, and the provider's id goes in a separate `responseModel` field (`packages/ai/src/types.ts:558`).
- Cost is still taken from the matching `allowedFallbackModels[].cost` (`packages/ai/src/api/anthropic-messages.ts:669-677`).
- Tests at `anthropic-sse-parsing.test.ts:142,169`.
- CHANGELOG `packages/ai/CHANGELOG.md:211`.

**Fix · [[opencode]]** `aa599b4a7d` 2026-01-21: the same-model check compared `model.api.id`, so legacy (pre-variant) model ids looked foreign and lost their metadata; switched to `model.id` (`packages/opencode/src/session/message-v2.ts:255`).

**Lesson** — Replay decisions must use the requested identity. Record the identity the server echoes separately, for display and pricing.

Related: [[cross-provider-handoff]] · [[signed-reasoning-replay]] · [[usage-cost-accounting]] · [[server-side-refusal-fallback]] · [[fallback-model-output-misattributed]] · [[pi--cross-provider-handoff|pi]] · [[opencode--cross-provider-handoff|opencode]]
