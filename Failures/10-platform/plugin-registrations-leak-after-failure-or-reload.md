---
type: failure
concepts: [runtime-plugin-loading]
harnesses: [pi]
---
**Symptom** — (a) An extension factory that threw half-way left its earlier registrations (event subscriptions, providers, flags) active (#8424). (b) Inter-extension event-bus listeners survived `/reload`, so old-generation handlers kept firing alongside new ones (#7656).

**Root cause** — Registration had immediate side effects with no transaction; reload invalidated the runner but not listeners attached to the shared `pi.events` EventEmitter.

**Fix · [[pi]]**
- `a69bef789` 2026-08-23 — two-phase load: factory runs against throwing action stubs, registrations staged on the `Extension` record, runtime changes queued while loading; `commit()` on success, `discard()` on throw rolls back subscriptions/providers/flags (`packages/coding-agent/src/core/extensions/loader.ts:244-268,532-548,613-632`).
- `6ca423447` 2026-08-05 — event-bus unsubscribers tracked per extension and cleared on invalidation (`loader.ts:162,196-210,523`).

**Lesson** — Plugin loading must be transactional and every subscription must be owned by a generation that is torn down as a unit.

Related: [[runtime-plugin-loading]] · [[extension-event-hooks]] · [[pi--runtime-plugin-loading|pi]]
