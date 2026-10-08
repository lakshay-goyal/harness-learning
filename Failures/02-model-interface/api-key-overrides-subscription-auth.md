---
type: failure
concepts: [credential-resolution, subscription-oauth-auth, secret-handling]
harnesses: [pi]
---
**Symptom** — Users logged in with a subscription (OAuth) were billed pay-as-you-go because an API key in `settings.json` took precedence over the stored OAuth credential.

**Root cause** — Credential resolution order favored config-file keys over stored subscription credentials; the precedence was implicit and the active auth mode wasn't surfaced.

**Fix · [[pi]]** — `a965b6f16` 2025-12-24 "Release v0.27.8 - OAuth takes priority over settings.json API keys" (`packages/coding-agent/CHANGELOG.md:5309`); later the active subscription is surfaced (0.66.0 warning; `warnings.anthropicExtraUsage` default true, `packages/coding-agent/src/core/settings-manager.ts:91`). Related leak fixes: [[env-credential-discovery-misfires]] (#4342), [[process-env-mutated-for-auth]] (#164), [[config-value-indirection-ambiguity]] (`9e9fc7947`).

**Lesson** — Credential resolution order is a billing and safety contract: make it explicit, prefer the credential the user most recently and deliberately configured, and show which one is active.

Related: [[credential-resolution]] · [[subscription-oauth-auth]] · [[secret-handling]] · [[pi--credential-resolution|pi]]
