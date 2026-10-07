---
type: failure
concepts: [http-transport-hardening]
harnesses: [pi]
---
**Symptom**
- Proxied plain-HTTP provider requests hung after a tool call.
- On Node 26, `response.json()` failed on compressed responses.
- User-supplied fetch overrides were clobbered.

**Root cause**
- Undici 8.7 switched HTTP-over-proxy from CONNECT tunnelling to non-tunnel forwarding.
- Node's bundled fetch combined with the npm undici dispatcher skipped decompression.

**Fix · [[pi]]**
- `a93f06660` 2026-06-16 — `undici.install()` makes global fetch and the dispatcher share one undici, unless the caller already replaced fetch (`packages/coding-agent/src/core/http-dispatcher.ts:101-112`).
- `23842b1e6` 2026-09-02 — `proxyTunnel: true`, "Keep HTTP origins on CONNECT tunnels as before Undici 8.7" (#8134) (`:86-91`).
- Global `EnvHttpProxyAgent` with `allowH2:false` (`:45-50,86-91`).

**Lesson** — Pin transport semantics across dependency upgrades, and keep fetch and dispatcher from the same implementation.

Related: [[http-transport-hardening]] · [[pi--http-transport-hardening|pi]] · [[transport-defaults-kill-connections]]
