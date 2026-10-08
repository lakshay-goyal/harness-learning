---
type: implementation
harness: opencode
concept: cross-provider-handoff
commit: ecc4916b5a
files: [packages/opencode/src/session/message-v2.ts:255, packages/opencode/src/session/message-v2.ts:288-294, packages/opencode/src/session/message-v2.ts:375-389, packages/core/src/session/runner/to-llm-message.ts:70-87, packages/llm/src/protocols/openai-chat.ts:210-331, packages/core/src/session.ts:402-415]
---
[[cross-provider-handoff]] in [[opencode]].

## Mechanism

### Legacy runtime
- Same-model test per stored assistant message: `differentModel = "${model.providerID}/${model.id}" !== "${msg.providerID}/${msg.modelID}"` (`packages/opencode/src/session/message-v2.ts:255`).
- Different model → text parts lose `providerMetadata`, tool parts lose `callProviderMetadata`, reasoning parts become **plain text** (empty ones dropped) (`message-v2.ts:288-294,375-389`). Same model → reasoning replayed with metadata (signatures).
- Rewrite happens on every request at `toModelMessages(msgs, targetModel)`; raw history untouched.

### v2 runtime
- Native Continuation Metadata reused only for the exact same provider + model id, and only if that assistant message had no error; otherwise non-empty reasoning → plain assistant text, metadata dropped (`packages/core/src/session/runner/to-llm-message.ts:70-87`). "May widen only when recorded provider tests establish compatibility" (`CONTEXT.md:135`).
- Model/agent switch applies at the next provider turn, never restarts the current one, and preserves the current Context Epoch (`CONTEXT.md:123,134`). `switchModel` ignores a no-op switch (same provider, id and variant) and otherwise publishes a durable `ModelSwitched` event (`packages/core/src/session.ts:402-415`); `model-switched` entries are dropped when lowering to provider messages.
- Agent binding: the agent is selected once per provider turn (`packages/core/src/session/runner/llm.ts:182`) and that `agent.id` is passed into every tool settlement of the turn (`:261-266`); "a later agent switch cannot change that call's policy" (`CONTEXT.md:125`) → [[opencode--permission-ruleset]].
- Hosted tools keep call-side and settlement-side provider metadata separately so settlement and interruption recovery cannot erase continuation ids (`specs/v2/session.md:52`; `specs/v2/schema-changelog.md:420-435`).
- OpenAI Chat lowering: reasoning concatenated into `reasoning_content`; tool-result images pulled out into a following `user` message because the `tool` role takes only text (`packages/llm/src/protocols/openai-chat.ts:210-331`).

## Constants
none.

## Evolution
- 2025-08-01 `8f45a0e227` Kimi K2 ⇄ Claude trajectory handoff.
- 2026-01-20 `021e42c0bb` foreign reasoning/metadata from another provider/account caused 400s → omit metadata when the model differs.
- 2026-01-21 `aa599b4a7d` comparison switched from `model.api.id` to `model.id` (legacy pre-variant ids regressed).
- 2026-02-01 `d1d7447493` Copilot: send `reasoning_text` only with `reasoning_opaque` when switching Anthropic models.

## Quirks / drift
- Same-model key is the configured model id, not the response model.

Failures: [[foreign-reasoning-signature-replayed]] · [[model-relabel-breaks-same-model-check]] · [[reasoning-not-replayed-degrades-tool-args]].

Contrast: [[pi--cross-provider-handoff|pi]] also converts foreign reasoning to untagged text in one `transformMessages` pass; opencode does the same in `toModelMessages`, then a second provider pass in `ProviderTransform.message`.
