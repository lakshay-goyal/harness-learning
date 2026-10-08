---
type: absence
harnesses: [codex]
---
# no-chat-completions-wire

Only the OpenAI Responses API is spoken; Chat Completions support (2025-05 → 2026-02) was deprecated and deleted.

**What's missing**
- Config `wire_api = "chat"` and provider `ollama-chat` fail deserialization with `CHAT_WIRE_API_REMOVED_ERROR` and a migration message (`codex-rs/model-provider-info/src/lib.rs:100-133`).

**Evidence of decision**
- Added `e924070cee` 2025-05-08 (Responses came first: `b940adae8e` same day).
- Deprecation announced (discussion #7782) with notice `43e6e75317` 2025-12-11 ("will be removed in early Feb 2026").
- Removed `d2394a2494` 2026-02-03 "chore: nuke chat/completions API (#10157)" and `88598b9402` "drop wire_api from clients".
- Quirk-mapping bugs on Chat Completions proxies (failure [[stop-reason-mapping-gaps]]) motivated the cut.

**Implication**
- Third-party models only via Responses-compatible endpoints/proxies (Ollama / LM Studio Responses, built-in Bedrock `cefcfe43b9` 2026-04-20 with AWS SigV4 `1cd3ad1f49`); no provider-neutral message model — opposite of pi's [[unified-provider-api]] breadth ([[provider-breadth]]).

Related: [[unified-provider-api]] · [[provider-breadth]] · [[custom-provider-registration]] · [[no-plugin-providers]] · [[Absences]]
