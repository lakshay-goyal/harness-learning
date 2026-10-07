---
type: failure
concepts: [signed-reasoning-replay]
harnesses: [pi]
---
**Symptom** — Anthropic-compatible providers lost reasoning continuity: pi downgraded their thinking to plain text on replay. Affected endpoints include Fireworks, Vercel AI Gateway, Xiaomi, Kimi and OpenCode qwen. On Bedrock, unsigned Claude thinking was rejected.

**Root cause** — "Compatible" endpoints disagree on what a signature is. Some emit empty signatures and accept them back as `signature:""`. Anthropic itself rejects unsigned thinking blocks. pi always converted blank-signature thinking into text.

**Fix · [[pi]]**
- `458a7bc27` (2026-05-28), #4464: opt-in compat flag `allowEmptySignature` replays `{type:"thinking", signature:""}` instead of text (`packages/ai/src/api/anthropic-messages.ts:1416-1431`). Default is false (`packages/ai/src/types.ts:914-976`). It is set for Fireworks (`packages/ai/scripts/generate-models.ts:1686`) and for all Vercel AI Gateway models (test `anthropic-empty-thinking-signature-compat.test.ts:101`).
- `c1449660c` (2026-09-28), #10047: enabled for OpenCode qwen3.8-flash.
- Vercel unsigned thinking replayed as text: #9676 (0.86.0).
- `c961cda2c` (2026-03-14), #2063: on Bedrock, Claude thinking with no signature is sent as plain text (`packages/ai/src/api/bedrock-converse-stream.ts:1040-1068`).

**Lesson** — Signature semantics differ across "compatible" endpoints. Model them with a per-model compat flag rather than one global rule.

Related: [[signed-reasoning-replay]] · [[model-catalog]] · [[signed-empty-reasoning-dropped]] · [[pi--signed-reasoning-replay|pi]]
