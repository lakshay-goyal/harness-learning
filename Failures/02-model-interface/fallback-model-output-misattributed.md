---
type: failure
concepts: [server-side-refusal-fallback, usage-cost-accounting]
harnesses: [pi]
---
**Symptom** After an Anthropic server-side fallback to another model, cost was computed at the requested model's price. A fallback block arriving after output had already started would have spliced two models' output into one message.

**Root cause** Cost was keyed to the requested model, and the parser accepted a `fallback` content block at any position.

**Fix · [[pi]]**
- Fallback pricing went through a four-commit saga. `a6c6f8018` 2026-08-18 was reverted by `59a71b235` 2026-08-18. Then `4809c2abc` 2026-08-19 and `ed867e909` 2026-08-19 ("fallback cost not via stream options; always pass beta header").
- When `message_start.model !== model.id`, `responseModel` is set and cost comes from the matching `allowedFallbackModels[].cost` (`packages/ai/src/api/anthropic-messages.ts:669-677`).
- `4e69b0c28` 2026-09-02: a `fallback` block with no output yet → `continue`. After output it throws `Anthropic performed an unsupported mid-output model fallback` (`anthropic-messages.ts:690-695`).
- Identity rule is in [[model-relabel-breaks-same-model-check]].

**Lesson** Price by the model that actually served the request. Fail loudly on server behavior you cannot represent, rather than splicing outputs.

Related: [[server-side-refusal-fallback]] · [[usage-cost-accounting]] · [[cross-provider-handoff]] · [[usage-priced-at-wrong-rate]] · [[pi--server-side-refusal-fallback|pi]]
