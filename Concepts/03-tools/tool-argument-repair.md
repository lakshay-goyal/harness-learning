---
type: concept
stage: tool-design
tier: candidate
aliases: [prepareArguments, prepareEditArguments, model-arg-repair, lenient-tool-arg-validation, validateToolArguments coercion]
harnesses: [pi]
---
A pre-validation shim that coerces common malformed argument shapes (stringified JSON, single object for array, legacy field names, numeric strings, stray nulls) into the canonical schema instead of rejecting the call.

## Why
- Each rejected call costs a full turn and can push the model to a worse tool (pi: models fell back to `sed`/python edits when `edits` arrived as a JSON string) — [[tool-arg-shape-drift]].
- Schema changes break resumed sessions whose history contains old-shape calls; keeping legacy shapes in the public schema re-introduces dual-shape confusion ([[edit-tool-dual-mode-confusion]]).
- Strict-mode providers make optional fields nullable → models send `null` that a non-strict validator rejects.

## Design space
- Reject with a descriptive error and let the model retry (baseline; pi still does this for unrepairable input, echoing received args).
- **Per-tool shim before validation** (pi `prepareArguments`) — keeps legacy/variant shapes out of the public schema.
- **Generic schema-driven coercion** (pi-ai: numeric strings, union-arm-first, null-on-optional dropped).
- Lenient schemas (`additionalProperties` allowed) vs strict (pi relaxed edit items after extra fields broke valid edits).
- Upstream prevention via constrained decoding → [[constrained-tool-sampling]]; downstream salvage of malformed streamed JSON → [[streaming-json-repair]].
- Reject out-of-range values with explanation rather than silently clamping (pi bash timeout).
- Mutate args in place vs repair on a copy (pi coding-agent mutates; pi durable copies).

## Implementations
- [[pi--tool-argument-repair|pi]] — `prepareArguments` hook (edit: JSON-string edits, single object, legacy oldText/newText) + pi-ai `validateToolArguments` coercion.

## Failures
- [[tool-arg-shape-drift]]
- [[tool-arg-coercion-breaks-unions]]
- [[edit-tool-dual-mode-confusion]]
- [[bash-timeout-clamped-to-immediate]]

## Related
[[tool-description-design]] · [[tool-error-as-result]] · [[constrained-tool-sampling]] · [[streaming-json-repair]] · [[search-replace-edit]] · [[session-migration]]
