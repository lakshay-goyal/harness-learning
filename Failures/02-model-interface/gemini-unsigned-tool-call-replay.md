---
type: failure
concepts: [cross-provider-handoff, signed-reasoning-replay]
harnesses: [pi]
---
**Symptom** — Gemini 3 rejected histories that contained function calls without a `thoughtSignature`, for example after a Claude→Gemini switch. Each workaround caused a new problem: text conversion lost structured context and invited mimicry, and the sentinel was rejected by Vertex.

**Root cause** — Gemini 3 in thinking mode requires a signature on every function call. Calls from foreign providers never have one, and sibling endpoints (Gemini API vs Vertex) do not accept the same workaround.

**Fix · [[pi]]** — a four-step saga:
1. `b18f401d9` (2026-01-15), #741: unsigned calls converted to text `[Tool Call: …]` (`google-shared.ts:135-156` at the time).
2. `5d3e7d5aa` (2026-01-16): text kept, plus an anti-mimicry note ("Historical context… do not mimic").
3. `a0d839ce8` (2026-03-05), #1829: send structured calls with the official sentinel `skip_thought_signature_validator` (`google-shared.ts:48,146-160` at the time).
4. `f7df47408` (2026-04-30), #4032: sentinel removed because Vertex rejects it. Calls are now sent unsigned as a plain `functionCall`.

At HEAD, `resolveThoughtSignature` (`packages/ai/src/api/google-shared.ts:146-160`) keeps a signature only for the same provider and model, and only when it is valid base64. Tests at `packages/ai/test/google-shared-gemini3-unsigned-tool-call.test.ts:100-128` assert that no sentinel is sent.

Follow-ups in the same arc:
- `cbaca6038` (2026-08-03), #7494: keep Gemini ≥3 tool-call ids → [[tool-call-id-requirement-drift]].
- `6138f5a07` (2026-07-31), #7362: keep signed empty parts → [[signed-empty-reasoning-dropped]].

**Lesson** — Signed-reasoning protocols make history non-portable. Each cross-model strategy (text, note, sentinel) failed on some backend. Turning calls into text "lobotomizes" the context, and vendor sentinels may not carry over to sibling endpoints.

Related: [[cross-provider-handoff]] · [[signed-reasoning-replay]] · [[thinking-tag-mimicry]] · [[foreign-reasoning-signature-replayed]] · [[pi--signed-reasoning-replay|pi]]
