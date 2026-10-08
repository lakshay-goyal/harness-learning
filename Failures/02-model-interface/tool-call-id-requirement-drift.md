---
type: failure
concepts: [tool-call-id-normalization, cross-provider-handoff]
harnesses: [pi]
---
**Symptom** — Gemini 3 history lost the pairing between function calls and function responses, so replayed calls and results no longer matched.

**Root cause** — `4f757fbe2` (2026-01-12) removed tool-call `id` fields from Google requests as "unsupported" because Vertex rejected them. Gemini 3 then started requiring ids. `requiresToolCallId` still excluded Gemini 3, so ids were stripped on replay.

**Fix · [[pi]]**
- `cbaca6038` (2026-08-03), #7047/#7494: `requiresToolCallId` now returns true for `claude-*`, `gpt-oss-*`, and Gemini whose parsed major version is ≥3. The version is parsed with `/^gemini(?:-live)?-(\d+)/` (`packages/ai/src/api/google-shared.ts:165-178`).
- Ids are normalized only when required: `id.replace(/[^a-zA-Z0-9_-]/g,"_").slice(0,64)` (`google-shared.ts:195-198`).
- `453b22397` (2026-03-18), #2052 is a related lesson: parse the version number instead of using `includes("gemini-3")`.

**Lesson** — Id requirements change between model generations. Gate them on the parsed model version, not on the provider. A field removed for one endpoint can become mandatory later.

Related: [[tool-call-id-normalization]] · [[cross-provider-handoff]] · [[gemini-unsigned-tool-call-replay]] · [[tool-call-id-collision]] · [[pi--tool-call-id-normalization|pi]]
