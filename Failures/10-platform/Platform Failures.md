---
type: group
group: 10-platform
---
Failures whose primary concept is in [[Platform]].

## Plugin hooks
- [[plugin-hook-wall-clock-timeout]] — hook timeouts killed hooks legitimately waiting on humans/LLMs.
- [[unbounded-hook-continuation-loop]] — unconditional `continue:true` at the settle boundary loops forever (documented hazard).

## Plugin loading / distribution
- [[stale-plugin-context-after-session-replacement]] — captured `ctx` silently targeted the old session after new/fork/switch/reload.
- [[plugin-registrations-leak-after-failure-or-reload]] — throwing factory / reload left registrations and bus listeners alive.
- [[plugins-fail-to-load-in-compiled-binary]] — TS plugins couldn't resolve host packages in Bun/SEA binaries.
- [[duplicate-host-module-instances]] — packages bundled their own copies of host libraries.
- [[module-cache-retains-plugin-generations]] — ESM cache leaked every hot-reloaded plugin generation.
- [[reload-cannot-drain-in-flight-calls]] — "drain then swap" reload contract couldn't be honored.
- [[plugin-tool-without-schema-breaks-requests]] — schema-less plugin tool broke every provider request.
- [[plugin-shortcut-shadows-core-keys]] — plugin shortcuts captured submit/interrupt/exit.

## Headless / events / TUI / export
- [[headless-protocol-stream-corruption]] — U+2028 splits, stray stdout writes, missing ids, backpressure loss broke RPC/JSON streams.
- [[quadratic-event-stream-output]] — cumulative snapshots per delta made JSON output O(n²).
- [[full-redraw-replays-history]] — full redraws duplicated scrollback (Termux height toggles).
- [[render-output-exceeds-string-limit]] — giant frames hit V8 max string length.
- [[render-storm-during-streaming]] — per-event renders burned CPU and delayed input.
- [[terminal-state-leaks-on-exit]] — raw mode / key events / SSH session corrupted on exit paths.
- [[terminal-input-sequence-fragmentation]] — split/batched escape sequences dropped keys, split Alt+Enter interrupted, Shift+Enter undetectable.
- [[exported-session-xss]] — exported HTML transcripts executed injected links/markup.
- [[concurrent-settings-writes-clobber]] — concurrent pi instances / invalid JSON clobbered settings.json.

## Client/server split & remote env
- [[unix-socket-endpoint-pitfalls]] — socket path length, inode reuse, hidden temp entries.
- [[server-worker-lifecycle-races]] — attach/lease/termination races across server, worker, client.
- [[relay-reconnect-blocked-by-close-code]] — illegal WebSocket close code threw and stopped relay reconnect.
- [[rewrite-drops-test-coverage]] — protocol rewrite deleted its tests.
- [[bulk-output-starves-control-messages]] — bulk data delayed pings/cancels over slow links.
- [[stale-handle-reaches-new-daemon]] — handles from a dead daemon could hit a fresh one.
- [[remote-bootstrap-hang]] — ssh/deploy/upload steps hung without timeouts or EOF.
- [[remote-platform-misdetection]] — garbled POSIX probe on Windows not recognized.
- [[remote-errno-parity-drift]] — remote errors diverged from Node semantics on Windows.
- [[file-watch-platform-gaps]] — FSEvents startup gap, Windows watchers block renames, NTFS identity.

## Evals & telemetry
- [[eval-harness-installs-registry-copy]] — evals installed pi from the registry and broke releases.
- [[eval-assertions-conflated-with-scores]] — setup breakage indistinguishable from low scores.
- [[failed-run-diagnostics-lost]] — cleanup errors discarded usage/transcripts.
- [[bespoke-shuffle-order-bias]] — custom seeded shuffle vs counterbalanced order.
- [[treatment-transform-marker-drift]] — prompt restructure silently disabled the control transform.
- [[validation-reads-projected-not-sent-prompt]] — validated stored prompt, not the sent one.
- [[control-arm-docs-leakage]] — control arm could still read the docs on disk.
- [[eval-judges-readable-by-agent]] — agent could read judge modules via SSR caches.
- [[conformance-suite-platform-timing]] — portable conformance cases timed out / leaked on some platforms.
- [[telemetry-docs-outlive-code]] — docs advertise deleted telemetry schemas and a missing `/privacy` command.

## Development process
- [[shared-worktree-agents-clobber-each-other]] — parallel agent sessions in one checkout committed/stashed/reset each other's work.

## Cross-group failures touching platform concepts
- [[tool-result-hook-patches-lost]] (Tools) — multiple hook handlers overwrote each other.
- [[side-door-input-bypasses-hooks]] (Safety) — RPC/custom-tool paths skipped extension hooks.
- [[context-handler-drops-system-state]] (Context) — `context` hook dropped system prompt + tool declarations.
- [[hook-throw-aborts-parallel-batch]] (Tools) — hook throw aborted a whole parallel batch.
- [[transport-defaults-kill-connections]] · [[stream-stall-without-header-timeout]] · [[node-only-imports-break-browser-bundle]] · [[env-credential-discovery-misfires]] (Model Interface) — runtime/HTTP platform issues.
- [[windows-process-tree-and-shells]] (Tools) — Windows process/shell primitives.
- [[pre-tool-hook-sees-stale-state]] (Safety) — `tool_call` hooks read stale session state in multi-tool turns.
- [[hook-error-fails-open]] (Safety) — failed `user_bash` routing hook ran the command on the local shell.
- [[strict-json-undefined-breaks-replication]] (State) — an `undefined` field blocked terminal transcript updates over the wire.
- [[liveness-timeout-counts-frames-not-bytes]] (Safety) — slow large frame could trip the liveness timeout (inferred).
- [[auth-at-message-layer]] · [[internal-errors-leak-over-wire]] · [[remote-binary-trusted-by-version-name]] (Safety) — client/server auth layer, error sanitization, remote daemon deploy trust.
- [[update-before-snapshot-on-subscribe]] · [[unbounded-subscriber-buffering]] · [[held-draft-reference-misaddresses]] · [[settled-draft-then-probe-throws]] · [[delta-tracking-memory-and-size-blowup]] (State) — Chord replicated-state wire/tracker failures under the client/server split.
