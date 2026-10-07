---
type: failure
concepts: [credential-resolution]
harnesses: [pi]
---
**Symptom**
- On Windows, literal API keys were corrupted: plain strings were treated as env-var names, and Windows' case-insensitive env matched them.
- Plain uppercase values were mistaken for env references.
- `!command` keys in models.json were resolved once and cached forever, so expired tokens kept being sent.

**Root cause** — Config values were implicit: "is this a literal or an env-var name?" was guessed. Command output was cached with no TTL into long-lived model state.

**Fix · [[pi]]**
- `def9e4e9a` 2026-01-18 — shell-command API keys in models.json (#762).
- `9cf5758b6` 2026-02-04 — shell commands and env vars in auth.json keys.
- `7a786d88a` 2026-03-27 — models.json auth resolved per request with no caching; TTL policy left to the user's wrapper command (#1835) (`packages/coding-agent/src/core/provider-composer.ts:467,477`; `resolve-config-value.ts:221-227`).
- `3e9f71744` 2026-05-28 — explicit `$VAR`/`${VAR}` syntax (#5095).
- `9e9fc7947` 2026-06-13 — plain uppercase values are literals (#5661).
- HEAD grammar: `!cmd` → stdout (execSync, 10 s timeout); `$$` → `$`; `$!` → `!`; else literal (`packages/coding-agent/src/core/resolve-config-value.ts:28-86,138-151,185-196`).
- Remaining asymmetry: auth.json commands are cached for the process lifetime, and a failure leaves the key unresolved until restart (`:9-10,208-216`; `packages/coding-agent/docs/providers.md:84`). models.json commands are uncached (`packages/coding-agent/docs/models.md:64`).

**Lesson** — Use explicit sigils for indirection in secrets config. Don't impose a TTL guess on user-supplied credential commands.

Related: [[credential-resolution]] · [[pi--credential-resolution|pi]] · [[env-credential-discovery-misfires]]
