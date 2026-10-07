---
type: group
group: 08-state
---
Failures whose primary concept is in [[State]].

- [[settled-tool-vanishes-before-placement]] — durable harness snapshot dropped committed-but-unplaced tool results; tool_end preceded its commit.
- [[tool-result-persisted-before-tool-call]] — async handlers wrote tool results to the log before their assistant tool-call message.
- [[torn-log-tail-fuses-next-entry]] — crash left an unterminated last line; next append fused with it; invalid files overwritten (pi, codex).
- [[session-lost-before-first-response]] — exiting during the first turn lost the session and the user's prompt.
- [[session-config-not-restored-on-resume]] — resume reset thinking level; clamped/session choices persisted wrongly.
- [[fork-boundary-loss]] — forks orphaned subtrees under labels and lost the compaction boundary.
- [[fork-writes-wrong-file]] — fork wrote into the parent file / duplicated headers / forked before turn settled.
- [[quadratic-long-session-operations]] — path walks, per-message lookups and whole-file loads blew up on long sessions.
- [[time-ordered-id-prefix-collision]] — short ids sliced from UUIDv7 timestamp prefixes collided.
- [[per-request-projection-rescans-log]] — durable runtime re-derived the whole transcript on every request.
- [[storage-queue-close-races]] — durable async SQLite queue/close races.
- [[output-window-depends-on-commit-cadence]] — durable tail-retained tool output varied with progress-commit timing.
- [[strict-json-undefined-breaks-replication]] — `undefined` property blocked remote transcript updates.
- [[held-draft-reference-misaddresses]] — held draft references wrote the wrong element after unshift/splice.
- [[settled-draft-then-probe-throws]] — revoked draft proxies threw on Promise `then` probes.
- [[delta-tracking-memory-and-size-blowup]] — proxy caches retained GiBs; 4096-op cap turned small changes into 66 MB snapshots.
- [[unbounded-subscriber-buffering]] — replicated-state subscribers buffered without bound / reentrant publication (pi); inverse — bounded event queue blocked request responses (codex).
- [[update-before-snapshot-on-subscribe]] — remote update arrived before the snapshot it applies to (pi); events lost between `/event` connect and lazy subscription (opencode).
- [[revert-deletes-preexisting-files]] — undo deleted user files missing from the shadow snapshot (opencode).
- [[host-vcs-config-leaks-into-shadow-repo]] — GPG signing, external diff, ignores and big repos leaked into the private snapshot repo (opencode).
- [[strict-schema-rejects-legacy-records]] — tightened stored-data schemas rejected old sessions; snapshot decode died as a defect (opencode).
- [[projector-depends-on-transitional-table]] — shared `Moved` projector touched a v2-only table on v1 databases (opencode).
- [[stale-runner-recreates-context-after-move]] — old-location runner could initialize stale privileged context after a move (opencode).
- [[reordered-context-misidentifies-latest-turn]] — "latest" by array position / id after a reordering projection → double compaction (opencode).

- [[fork-carries-startup-context]] — (codex) last-N fork returned startup context; first-turn model-switch text survived rollback.
- [[forked-child-inherits-parent-tool-noise]] — (codex) full-history sub-agent fork copied parent reasoning/tool calls/role instructions.
- [[index-db-corruption]] — (codex) SQLite index corrupted by WAL-reset bug; now backup + rebuild from rollouts, quick_check at open.
- [[turn-items-lost-on-abort]] — (codex) turn items appended only on success; Ctrl-C lost the whole turn.
- [[unbounded-payload-in-transcript]] — (codex) multi-hundred-MB MCP results bloated rollout JSONL; persistence-only 64 KiB caps.
- [[undo-clobbers-user-git-state]] — (codex) ghost-commit `/undo` restored with `--staged`, wiping the user's index; feature un-shipped.

Other groups' failures touching state concepts: [[abandoned-attempts-left-in-context]] · [[session-switch-leaves-dangling-tool-calls]] · [[branch-summary-records-wrong-source-leaf]] · [[compaction-includes-abandoned-branches]] · [[stream-scratch-state-persisted]] · [[failed-turns-replayed]] · [[quadratic-event-queue-drain]] · [[non-idempotent-tool-replayed-after-crash]] · [[ownership-cancellation-races]]

Back: [[State]]
- [[timestamp-ordered-history-pagination]] — wall-clock ordering reordered history (double auto-compaction); v2 orders by durable seq.
