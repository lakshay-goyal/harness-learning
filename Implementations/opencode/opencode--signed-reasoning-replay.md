---
type: implementation
harness: opencode
concept: signed-reasoning-replay
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:20-23, packages/opencode/src/provider/transform.ts:501-516, packages/opencode/src/provider/transform.ts:693-740, packages/opencode/src/provider/transform.ts:1234-1247, packages/opencode/src/provider/transform.ts:303-354, packages/opencode/src/session/message-v2.ts:272-295, packages/opencode/src/session/processor.ts:438-451, packages/llm/src/protocols/openai-responses.ts:368-451, packages/llm/src/protocols/openai-responses.ts:991, packages/llm/src/providers/openai-options.ts:44-70]
---
[[signed-reasoning-replay]] in [[opencode]].

## Mechanism

### Legacy runtime
- **OpenAI Responses stateless**: `store: false` default for openai, `@ai-sdk/openai`, Copilot, Bedrock-mantle, xAI and Azure (`packages/opencode/src/provider/transform.ts:1234-1247`); gpt-5 family requests `reasoningSummary: "auto"` + `include: ["reasoning.encrypted_content"]` (`INCLUDE_ENCRYPTED_REASONING`, `transform.ts:20-23,978-1039`).
- Responses item ids are stripped from replayed parts when `store !== true` — "following Codex and keeping signed request bodies immutable", done in message transform **before** serialization/signing (`transform.ts:501-516`). Copilot ids stored under a `copilot` key and stripped too.
- **Anthropic**: same-model reasoning replayed with `signature`; empty text separator between signed thinking blocks becomes `" "` (`packages/opencode/src/session/message-v2.ts:272-289`); empty-but-signed/redacted reasoning kept by the empty filter (`transform.ts:182-188`).
- **Block binding** (Claude 5.1+, not "mythos" 5.1): request `thinking.blockBinding: {prefixMismatchBehavior: "drop_block"}` (Bedrock: `reasoningConfig`) because opencode re-renders system/tools/compaction between turns; config opt-out `blockBinding: false` (`transform.ts:693-740`). Dropped blocks reported by the API (`anthropic.inputTransformations`) are logged "thinking blocks dropped by provider" (`packages/opencode/src/session/processor.ts:438-451`). Needs the vendored `@ai-sdk/anthropic` / `@ai-sdk/amazon-bedrock` patches.
- **DeepSeek / interleaved field**: every assistant message gets a (possibly empty) reasoning part; for `capabilities.interleaved.field` models reasoning text goes into `providerOptions.openaiCompatible[reasoning_content | reasoning_details]`, even when empty (`transform.ts:303-354`).
- **Copilot**: `reasoning_opaque` paired with `reasoning_text`; Copilot Responses reasoning tracked by `output_index` (custom Copilot SDK under `packages/opencode/src/provider/sdk/copilot/`).

### v2 runtime (`packages/llm`)
- OpenAI Responses route default `providerOptions.openai.store = false` (`packages/llm/src/protocols/openai-responses.ts:991,1019`); GPT-5 (not `-chat`/`-pro`) defaults `reasoningEffort: "medium"`, `reasoningSummary: "auto"`, `include: ["reasoning.encrypted_content"]` — "the only way a follow-up turn can carry reasoning state" (`packages/llm/src/providers/openai-options.ts:44-70`).
- `store === false`: reasoning replayed inline `{type, summary, encrypted_content}` **without item id**; items lacking encrypted state filtered (`openai-responses.ts:368-399,441-451`). `store !== false`: reasoning and hosted tools replayed as `item_reference`.
- Native Continuation Metadata reused only for exact same model and non-errored turns (`packages/core/src/session/runner/to-llm-message.ts:70-87`).
- Anthropic lowering emits `{type:"thinking", thinking, signature: undefined}` when metadata was dropped for a failed same-model turn; no `redacted_thinking` lowering (`packages/llm/src/protocols/anthropic-messages.ts:449-455`) — provider acceptance unverified.

## Constants
| name | value | path:line |
|---|---|---|
| `INCLUDE_ENCRYPTED_REASONING` | `["reasoning.encrypted_content"]` | `packages/opencode/src/provider/transform.ts:23` |
| `ANTHROPIC_BLOCK_BINDING` | `{prefixMismatchBehavior: "drop_block"}` | `packages/opencode/src/provider/transform.ts:709` |

## Evolution
- 2025-09-14 `e3e459fc50` reasoning metadata persisted; 2025-09-17 `8c2aec43b8` → `3c3d6b65c2` same-day revert; 2025-09-26 `5d95846df1` re-landed ("reasoning without its required following item").
- 2026-01-19 `260ab60c0b` Copilot reasoning by `output_index`; 2026-02-01 `d1d7447493` Copilot opaque pairing.
- 2026-04-24 `86715fecc4` / `923af96d26` DeepSeek V4: reasoning on every assistant message, empty kept.
- 2026-04-29 `a740d2c667` Azure aligned; 2026-06-08 `a86ecf3bba` id stripping moved before request signing (fetch-hook rewrite broke Bedrock-mantle signatures); 2026-06-30 `f1407e41c4` stale Copilot ids.
- 2026-05-21 `61390dbb49` / 2026-06-26 `f254476043` v2 re-hit continuation-metadata and stateless-id bugs; 2026-07-24 `4b19ea2a71` message phases (+880) reverted same day `7840562d1b`.
- 2026-07-23 `20589d66d5` Mistral reasoning history via SDK patch.
- 2026-09-01 `3f39a329c3` tolerate block binding; 2026-09-02 `68abdce1a0` opt-out, `9a71624d2d` scoped to Claude 5.1+.

## Quirks / drift
- Request rewrites must happen before signing: a fetch-level body rewrite broke signed Bedrock requests.

Failures: [[responses-reasoning-item-pairing]] · [[opaque-reasoning-payload-lost]] · [[stale-thinking-signature-after-prefix-change]] · [[signed-empty-reasoning-dropped]] · [[reasoning-not-replayed-degrades-tool-args]] · [[foreign-reasoning-signature-replayed]].

Contrast: [[pi--signed-reasoning-replay|pi]] also replays stateless encrypted items and uses `drop_block`; opencode reached both through a year of reverts and SDK patches, then re-implemented them in `packages/llm`.
