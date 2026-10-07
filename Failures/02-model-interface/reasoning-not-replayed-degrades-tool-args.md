---
type: failure
concepts: [signed-reasoning-replay, cross-provider-handoff]
harnesses: [pi, opencode]
---
**Symptom** — Two failure shapes:
- Behavioral: Qwen via OpenAI-compatible endpoints degraded multi-turn tool-call arguments to empty `{}`.
- Hard errors: DeepSeek V4, Xiaomi MiMo, Kimi (OpenCode Go) and Z.AI sessions returned 400s or lost cache after turns replayed without reasoning.

**Root cause** — Prior-turn thinking was dropped from replay. Open-weight chat templates need it to keep behaving the same way. Some APIs require `reasoning_content` on every assistant message. Z.AI clears thinking unless told otherwise, which busts the cached prefix.

**Fix · [[pi]]**
- `e3f6912d4` (2026-04-17), #3325: `chat_template_kwargs:{enable_thinking:true, preserve_thinking:true}` for `qwen-chat-template` (`packages/ai/src/api/openai-completions.ts:901-905`).
- `9b103e5e4` (2026-04-24), #3636/#3668: replay DeepSeek V4 reasoning, plus the compat flag `requiresReasoningContentOnAssistantMessages`. Assistant messages lacking it get `reasoning_content:""` when compat `requiresReasoningContentOnAssistantMessages` is set and the model reasons (`packages/ai/src/api/openai-completions.ts:1386-1392`). MiMo: #4678.
- `21d80deda` (2026-05-18), #4251: opencode-go normalizes `reasoning` → `reasoning_content` on both stream and replay (`:625-628,1343-1345`).
- `b91bdd5a3` (2026-06-29), #6083: Z.AI `thinking:{type:"enabled", clear_thinking:false}` (`:880-892`).
- `a37306d43` (2026-10-05): Azure Foundry keeps `reasoning_content` on assistant turns so the cached prefix stays byte-identical (tests `packages/ai/test/azure-openai-completions.test.ts:111-249`).
- Replay field selection: the signature names the field the provider used (`reasoning_content`/`reasoning`/`reasoning_text`) (`:1340-1349`).

**Fix · [[opencode]]** `86715fecc4` / `923af96d26` 2026-04-24: DeepSeek V4 thinking mode rejected assistant history lacking `reasoning_content`, even empty; every assistant message now gets a (possibly empty) reasoning part, kept in the interleaved field (`packages/opencode/src/provider/transform.ts:303-354`). `e7053c41f4` 2026-04-26 OpenRouter SDK bump for DeepSeek reasoning; `6ca60d9204` 2026-07-01 Cerebras SDK reasoning replay.

**Lesson** — For open-weight chat templates, dropping prior reasoning changes model behavior, not just cost. Some providers need an explicit "preserve thinking" flag to keep cache-stable prefixes.

Related: [[signed-reasoning-replay]] · [[cross-provider-handoff]] · [[thinking-level-abstraction]] · [[pi--signed-reasoning-replay|pi]] · [[opencode--signed-reasoning-replay|opencode]]
