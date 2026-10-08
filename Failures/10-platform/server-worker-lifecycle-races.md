---
type: failure
concepts: [client-server-session-split]
harnesses: [pi]
---
**Symptom** — Experimental client/server sessions misbehaved around lifecycle edges: timed-out attaches left half-attached state, worker shutdown raced new leases, terminations raced concurrent opens; mini/remote clients lost operations and aborts; remote prompts returned before their terminal event.

**Root cause** — Multi-process lifecycle (server generation, per-session worker, client attachment) had no single reconciliation of demand, leases and termination.

**Fix · [[pi]]**
- `fc0a7cb36` 2026-08-16 — reconcile server lifecycle races: timed-out attach compensation, bounded shutdown, concurrent lease vs termination (+ conformance tests).
- `ee398cb6d` 2026-09-01 — recover mini operations and forward aborts; `b7f788194` 2026-09-19 — await terminal remote prompt event.
- HEAD design: `WorkerLifecycle` retires only with demand initialized, no holds, no live tasks, no attachments (`packages/coding-agent/src/experimental/session-worker.ts:293-305`); router serializes per-client ops, release waits for admitted ops, concurrent opens share one promise (`packages/server/src/session-router.ts:146-274`).

**Lesson** — Model process lifecycle as explicit demand/hold state with one reconciler; every timeout needs a compensating action.

Related: [[client-server-session-split]] · [[abort-propagation]] · [[ownership-cancellation-races]] · [[pi--client-server-session-split|pi]]
