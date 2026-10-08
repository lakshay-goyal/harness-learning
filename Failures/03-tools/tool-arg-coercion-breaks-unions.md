---
type: failure
concepts: [tool-argument-repair, streaming-json-repair]
harnesses: [pi]
---
**Symptom**
- Tool calls with numbers sent as strings failed validation.
- After coercion was added, nullable unions broke because `null` was coerced into another primitive.
- AJV validation crashed under CSP/eval restrictions (Cloudflare Workers).

**Root cause** — Lenient coercion was applied before checking whether the args were already valid, and the validator compiled code with `eval`.

**Fix · [[pi]]**
- `923b9cb9e` 2026-01-16 — coerce string numbers, cloning first (#786).
- `0d7c81ec9` 2026-03-19 — skip AJV validation in restricted runtimes (#2395).
- `2e95584da` 2026-08-03 — try matching union arms unchanged before coercing (#7328/#7373) (`packages/ai/src/utils/validation.ts:175-192`).
- HEAD `validateToolArguments` (`validation.ts:6,317-350`): structuredClone → `normalizeOptionalNulls` → TypeBox `Value.Convert` → compiled validator cached in a WeakMap. The error text (path: message lines plus the pretty-printed received args) is what the model sees (`:344-347`).

**Lesson** — Lenient coercion needs "already valid? leave it alone" as its first rule.

Related: [[tool-argument-repair]] · [[streaming-json-repair]] · [[tool-error-as-result]] · [[malformed-tool-json-crashes]]
