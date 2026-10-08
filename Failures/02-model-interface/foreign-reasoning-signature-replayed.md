---
type: failure
concepts: [signed-reasoning-replay, cross-provider-handoff]
harnesses: [pi, opencode]
---
**Symptom** — Google returned 400s when history was replayed after a model or provider switch. Non-Claude Bedrock models rejected replayed reasoning.

**Root cause** — Opaque reasoning tokens were sent to a provider or model that did not mint them:
- Google's `thoughtSignature` is a TYPE_BYTES field, so foreign or non-base64 strings fail validation.
- Bedrock's `reasoningText.signature` field is only accepted by Claude models.

**Fix · [[pi]]**
- `934e7e470` (2026-01-12), #654: keep signatures only for the same provider AND model AND valid base64. The check is `/^[A-Za-z0-9+/]+={0,2}$/` plus `length % 4 === 0` (`packages/ai/src/api/google-shared.ts:146-160`).
- `2c7c23b86` (2026-01-19), #821: the shared pass deletes `thoughtSignature` on any tool call that is not from the same model (`packages/ai/src/api/transform-messages.ts:131-134`). `isSameModel` means provider, api and model id are all equal (`:95-98`).
- `9a438465e` (2026-01-14), #727: non-Claude Bedrock models get `reasoningText{text}` without a signature (`packages/ai/src/api/bedrock-converse-stream.ts:1040-1068,894-904`).
- Redacted thinking is kept only for the same model; otherwise it is dropped (`transform-messages.ts:104-106`).

**Fix · [[opencode]]** `021e42c0bb` 2026-01-20: switching provider or account replayed reasoning signatures, item ids and encrypted content from the other model → 400. `toModelMessages` now drops provider metadata and turns reasoning into text when `providerID/model.id` differs (`packages/opencode/src/session/message-v2.ts:255,375-389`).

**Lesson** — Opaque reasoning tokens are valid only for the exact (provider, api, model) that minted them. Validate their format before echoing them back.

Related: [[signed-reasoning-replay]] · [[cross-provider-handoff]] · [[gemini-unsigned-tool-call-replay]] · [[model-relabel-breaks-same-model-check]] · [[pi--cross-provider-handoff|pi]] · [[opencode--cross-provider-handoff|opencode]]
