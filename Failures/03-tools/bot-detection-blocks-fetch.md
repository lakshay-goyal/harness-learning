---
type: failure
concepts: [web-tools]
harnesses: [opencode]
---
**Symptom** — `webfetch` got 403 Cloudflare challenges on many sites despite sending a Chrome User-Agent.

**Root cause** — The UA claimed Chrome while the TLS fingerprint was not a browser's; the mismatch is itself a bot signal (code comment).

**Fix · [[opencode]]** — `b978ca11da` 2026-01-24 "retry webfetch with simple UA on 403 (#10328)": on 403 with `cf-mitigated: challenge`, retry once with `User-Agent: opencode` (`packages/opencode/src/tool/webfetch.ts:84-88`). The Chrome 143 UA remains the first attempt (`:71`).

**Lesson** — A spoofed browser UA without a browser fingerprint can be worse than an honest UA; keep an honest fallback.

Related: [[web-tools]] · [[opencode--web-tools|opencode]]
