---
type: failure
concepts: [cache-stable-prompt-prefix, minimal-system-prompt, env-vars-as-context]
harnesses: [pi]
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

**Lesson** — Anything volatile in the prompt prefix costs the whole cache on every change; drop it or move it to env vars/tools/the user turn — narrowing it (time → date) only reduces how often you pay.

Related: [[cache-stable-prompt-prefix]] · [[minimal-system-prompt]] · [[env-vars-as-context]] · [[cache-miss-accounting]] · [[pi--cache-stable-prompt-prefix|pi]]
