---
type: failure
concepts: [cache-stable-prompt-prefix, minimal-system-prompt, env-vars-as-context]
harnesses: [pi, opencode]
---
**Symptom** — Prompt cache missed on every reload, resume, and (later) every new day: the full system prompt + history was re-billed at write/input price instead of cache-read price.

**Root cause** — The system prompt embedded the current time: `Current date and time: <locale string>` appended at the end of the prompt (`b1c2c32e2` 2025-11-12). Each step of the fix only narrowed the volatility: time → ISO date → locale-dependent date formatting → still a daily change. Anything volatile in the prefix changes every downstream cache key.

**Fix · [[pi]]**
- Precedent: `2a0f23928` 2025-12-11 (mom bot) "remove dynamic timestamp from system prompt for better cache hits".
- `4b9e6006f` 2026-03-13 (#2131): "Current date and time: Tuesday, …, 14:03:22 GMT" → "Current date: YYYY-MM-DD" — "so prompt prefixes stay cacheable across reloads and resumed sessions" (`packages/coding-agent/CHANGELOG.md:3124`).
- `f81acc667` (+`7f55605aa`) 2026-04-17 (#2814): date built from components, not locale — "keeping prompts deterministic across runtimes and locales" (CHANGELOG `:2454`).
- `f4e9ca746` 2026-07-14 (#6621, @davidbrai): date removed entirely — "Fixed system prompt cache invalidation across dates by removing the current date from the default prompt" (CHANGELOG `:1252`). HEAD: `git grep 'Current date'` empty in coding-agent/agent src → [[no-date-in-prompt]].
- Session facts moved to env vars instead (`bb3d7d399`, `bash.ts:186-212`) → [[env-vars-as-context]].
- Durable spec codifies it: "Renderers must be deterministic for equal inputs: any change in rendered text, such as an embedded timestamp, appends a system delta and invalidates provider prompt caches." (`packages/durable/docs/spec.md:3206-3208`); footgun "Unstable prompt text" (`spec.md:4718-4720`).

**Fix · [[opencode]]** Volatility lived in tool descriptions as well as the system prompt.
- `179c40749d` 2026-02-13 (#13559) websearch description embedded today's date → "The current year is {{year}}" (`packages/opencode/src/tool/websearch.txt:13`).
- `15a8c22a26` 2026-03-27 (#19487) bash description embedded the project directory → no cross-project cache hits; re-introduced by unrelated `b234370080` 2026-03-30 (PowerShell support); removed "again" `38014fe448` 2026-04-02 (#20771).
- Still at HEAD (legacy): `Today's date: ${new Date().toDateString()}` inside `<env>` (`packages/opencode/src/session/system.ts:83`) — the system prefix changes at midnight.
- v2 design fix `1af8dafd3e` 2026-06-04: date and environment are Context Sources; the baseline is frozen per Context Epoch and a change appends "Today's date is now: …" as a chronological update (`packages/core/src/system-context/builtins.ts:33-39`; `CONTEXT.md:130-132`) → [[opencode--transcript-carried-system-prompt]].

**Lesson** — Anything volatile in the prompt prefix costs the whole cache on every change; drop it or move it to env vars/tools/the user turn — narrowing it (time → date) only reduces how often you pay.

Related: [[cache-stable-prompt-prefix]] · [[minimal-system-prompt]] · [[env-vars-as-context]] · [[cache-miss-accounting]] · [[pi--cache-stable-prompt-prefix|pi]]
