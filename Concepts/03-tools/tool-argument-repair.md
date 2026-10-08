---
type: concept
stage: tool-design
tier: must-have
aliases: [prepareArguments, prepareEditArguments, model-arg-repair, lenient-tool-arg-validation, validateToolArguments coercion, experimental_repairToolCall, "invalid tool", InvalidArgumentsError, ParseMode::Lenient, PARSE_IN_STRICT_MODE]
harnesses: [pi, opencode, codex]
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
- Repair the free-text envelope rather than JSON shapes (✔ codex: strip `<<'EOF'` heredoc wrappers for all models, accept `applypatch`, re-route `apply_patch` heredocs sent through the shell).
- Clamp out-of-range waits instead of rejecting (✔ codex `yield_time_ms` → [250, 30000], short `wait_agent` timeouts → floor) vs reject with explanation (✔ pi bash timeout).
- Refuse with the exact corrected invocation in the error (✔ codex `patch detected without explicit call to apply_patch. Rerun as ["apply_patch", "<patch>"]`).
- Tolerate legacy enum values after they leave the schema (✔ codex `view_image.detail`).
- User hooks may rewrite arguments before dispatch (✔ codex PreToolUse `updated_input`).
- Name-only repair via the SDK repair hook + a never-offered sink tool (opencode).
- Move soft limits from the schema into the description (opencode question tool).

## Implementations
- [[pi--tool-argument-repair|pi]] — `prepareArguments` hook (edit: JSON-string edits, single object, legacy oldText/newText) + pi-ai `validateToolArguments` coercion.
- [[codex--tool-argument-repair|codex]] — lenient `apply_patch` parser + `applypatch` alias + shell interception; clamps for yield/wait; legacy value tolerance; PreToolUse input rewrite.
- [[opencode--tool-argument-repair|opencode]] — name repair only (lowercase match, else rewrite to a hidden `invalid` sink tool); schema failures → "Please rewrite the input so it satisfies the expected schema"; leniency by loosening schemas.

## Failures
- [[tool-arg-shape-drift]]
- [[tool-arg-coercion-breaks-unions]]
- [[edit-tool-dual-mode-confusion]]
- [[bash-timeout-clamped-to-immediate]]
- [[patch-wrapped-in-heredoc-by-model]]
- [[tool-name-training-artifact]]
- [[patch-body-executed-as-shell]]

## Related
[[tool-description-design]] · [[tool-error-as-result]] · [[constrained-tool-sampling]] · [[streaming-json-repair]] · [[search-replace-edit]] · [[session-migration]] · [[patch-envelope-edit]] · [[tool-schema-lowering]]
