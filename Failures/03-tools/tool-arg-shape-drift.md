---
type: failure
concepts: [tool-argument-repair, search-replace-edit]
harnesses: [pi]
---
**Symptom** — Calls with valid intent rejected by schema validation: `edits` sent as a JSON-encoded string (models then **abandoned the edit tool and fell back to `sed`/python** in bash); a single edit object instead of a one-element array; model-invented extra fields on edit items; `null` for omitted optional args; providers omitting tool inputs entirely; old-shape calls replayed from resumed sessions.

**Root cause** — Models (named in code: Opus 4.6, GLM-5.1) serialize nested arrays as strings or collapse singletons; strict-mode providers turn optionals into nullables; a strict `additionalProperties:false` schema rejects harmless extras.

**Fix · [[pi]]**
- Numeric-string coercion (`923b9cb9e`) and union-arm preservation (`2e95584da`) → separate note [[tool-arg-coercion-breaks-unions]].
- `af813f904` 2026-01-30 (#1065) — default `{}` when providers omit inputs.
- `b5f425ad1` 2026-03-29 — `prepareArguments` hook; legacy `oldText/newText` folded (`packages/agent/src/agent-loop.ts:694-707`).
- `a2ec01e12` 2026-04-18 (#3370) — parse stringified `edits` (`packages/coding-agent/src/core/tools/edit.ts:110-118`).
- `a1b336d73` 2026-07-04 (#6278) — edit item objects allow extra properties.
- `7915cdac6` 2026-08-11 — `normalizeOptionalNulls` drops `null` on optional non-nullable props (`validation.ts:239-269`).
- `ca21c1686` 2026-08-17 (#8011/#7835) — single edit object → array (`edit.ts:119-121`).
- `49681e1b7` 2026-09-25 (#9996) — read calls with `null` offset/limit no longer rendered as `:1`.

**Lesson** — Validate leniently at the boundary: coerce near-misses into the canonical shape, because every rejection costs a turn and can push the model to a worse tool.

Related: [[tool-arg-coercion-breaks-unions]] · [[tool-argument-repair]] · [[search-replace-edit]] · [[constrained-tool-sampling]] · [[pi--tool-argument-repair|pi]] · [[strict-tool-schema-rejections]]
