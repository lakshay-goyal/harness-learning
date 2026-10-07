---
type: failure
concepts: [tool-description-design, search-replace-edit]
harnesses: [pi]
---
**Symptom** — The write tool reported "Successfully wrote N bytes" where N was wrong for any non-ASCII content; the model could reason from the bogus size.

**Root cause** — `content.length` counts UTF-16 code units, not bytes.

**Fix · [[pi]]** — `e583b290a` 2026-09-02 (#8979) "remove misleading write byte counts": result is now `Successfully wrote to {path}` (`packages/coding-agent/src/core/tools/write.ts:86`; same fix in the experimental harness `packages/agent/src/harness/tools/write.ts:33` at `e583b290a` — that harness was removed in `7fd478a2e`).

**Lesson** — Don't put numbers in tool results that you don't compute exactly; the model treats every stated fact as true. (Same family: [[signal-killed-command-reported-success]], [[placeholder-text-misleads-model]].)

Related: [[tool-description-design]] · [[search-replace-edit]] · [[pi--search-replace-edit|pi]]
