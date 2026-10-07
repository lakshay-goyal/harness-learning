---
type: failure
concepts: [token-estimation, auto-compaction]
harnesses: [pi]
---
**Symptom** — Auto-compaction re-triggered immediately after a compaction; the first compaction entry was used as boundary instead of the latest; the footer showed stale context %.

**Root cause** — Messages kept verbatim across a compaction still carry the provider usage of the *old, larger* prefix; using that usage as "current context size" says the session is still over threshold.

**Fix · [[pi]]**
- `7eb969ddb` 2026-02-12 (#1382) use latest compaction entry (`getLatestCompactionEntry`), show `?/200k` until next real usage.
- `a4f4d91fa` 2026-03-05, `d5c18e024` 2026-03-06 (#1860), `d1a17bbae` 2026-03-06: ignore assistant usage older than the latest compaction in threshold and error paths (`packages/coding-agent/src/core/agent-session.ts:2966-2974`, `3058-3076` at HEAD).
- `8973ae28a` 2026-07-09 (#6464) pi-ai estimator ignores usage from responses older than a later-inserted prefix message (`packages/ai/src/utils/estimate.ts:71-95`).
- `466db0fec` 2026-09-21 `estimateProjectedContextTokens` trusts usage only if its entry is after the latest `compaction`/`context_edit` (`packages/coding-agent/src/core/compaction/compaction.ts:226-262`).

**Lesson** — Provider usage is valid only for the prefix it measured; tag it with its boundary and invalidate at every compaction or edit.

Related: [[token-estimation]] · [[auto-compaction]] · [[compaction-starved-by-missing-usage]] · [[output-token-cap-misbudgeted]] · [[overflow-judged-against-wrong-model]]
