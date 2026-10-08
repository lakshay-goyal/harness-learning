---
type: failure
concepts: [unified-provider-api]
harnesses: [pi]
---
**Symptom**
- The Codex terminal event was lost when it was not followed by a blank line. The turn looked truncated or hung (#9047).
- Proxies injected non-protocol SSE events (an OpenAI-style `done` on an Anthropic route), which crashed parsing.
- Line-ending variants broke frame splitting.

**Root cause** The hand-written SSE parsers split only on `\n\n`, never flushed the decoder at EOF, and accepted any event name.

**Fix · [[pi]]**
- `64eeb82a4` 2026-09-03: flush the decoder at EOF and treat the residual buffer as a frame (`packages/ai/src/api/openai-codex-responses.ts:799-859`) (#9047).
- `3e7ffff18` 2026-04-25: whitelist `message_start/delta/stop` and `content_block_start/delta/stop`, and ignore other events (`packages/ai/src/api/anthropic-messages.ts:391-398,546-548`). `iterateSseMessages` handles CR, LF, CRLF, `:` comments, multi-line `data` and a trailing flush (`anthropic-messages.ts:400-528`). It is pi-owned since `4b926a30a` (see [[malformed-tool-json-crashes]]).
- Mistral parser handles all CR/LF boundary combinations, multi-line `data:` and the `[DONE]` sentinel, and throws on non-`choices` JSON (`packages/ai/src/api/mistral-conversations.ts:491-511`).

**Lesson** A custom SSE parser must flush at EOF, accept every line-ending variant, and whitelist protocol event names at the boundary.

Related: [[unified-provider-api]] · [[terminal-event-required]] · [[truncated-stream-accepted-as-success]] · [[http-transport-hardening]] · [[pi--unified-provider-api|pi]]
