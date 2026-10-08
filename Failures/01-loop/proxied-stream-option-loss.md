---
type: failure
concepts: [turn-loop, unified-provider-api, auto-compaction]
harnesses: [pi]
---
**Symptom** — Model calls routed through an indirection layer behaved like a different agent: the agent-core `streamProxy` dropped session/transport/cache/thinking options (#3512) and Responses tool-call namespaces (#7709); compaction bypassed custom `streamFn`s (#4484); extension model calls lost credential-resolved endpoints, e.g. Copilot Business (0.84.0, #6768, #7579).

**Root cause** — Each indirection (proxy, side-call path, extension `complete`) re-built the stream option bag by hand instead of forwarding it whole.

**Fix · [[pi]]**
- `32859bdf9` 2026-04-22 — preserve proxy stream options (`packages/agent/src/proxy.ts`) (#3512).
- `35f807cfa` 2026-05-17 — route compaction through `streamFn` (`packages/coding-agent/src/core/compaction/compaction.ts`, `agent-session.ts`) (#4484).
- `02bd2d1c6` 2026-08-07 — preserve Responses tool-call namespaces through `proxy.ts` and server protocol (`packages/agent/src/proxy.ts:341`) (#7709).
- 0.84.0 — extension auth endpoints preserved (#6768, #7579; cf. `e741cb05c` 2026-08-04).

**Lesson** — Every indirection layer for model calls (proxy, side calls, subagents) must forward the full option bag, or side calls silently run with different identity, cache and reasoning settings.

Related: [[turn-loop]] · [[unified-provider-api]] · [[auto-compaction]] · [[compaction-request-shape-mismatch]] · [[credential-resolution]] · [[pi--turn-loop|pi]]
