---
type: implementation
harness: pi
concept: spec-driven-agentic-development
commit: b30a6dd77
files: [packages/durable/docs/pico-v5-handoff.md:1-917, packages/durable/docs/spec.md:1-4747, packages/durable/docs/pico-v5-chord-usage.md, packages/durable/docs/chord-delta-findings.md, packages/chord/PLANNING.md]
---
[[spec-driven-agentic-development]] in [[pi]].

## Mechanism
- Normative spec: `packages/durable/docs/spec.md` (4747 lines) — invariants (`:63-78`), tasks/effect sandwich (`:1609-2176`), built-in tasks (`:3297-3681`), storage (`:4320-4594`), structural footguns listed as contracts not guards (`:4596-4731`), non-goals (`:4741-4747`).
- Handoff for the implementer: `packages/durable/docs/pico-v5-handoff.md` (917 lines): "`packages/durable/docs/spec.md` is normative. Implement this list in order. After every package: run its tests, run `npm run check`, and stop for user review. Do not redesign later packages while implementing the current one." (`:3-5`). "Pico3 is reference material only. Preserve useful behavior, not its capability facades, membranes, document routing, view projection, events, or clone chains." (`:7-8`). Status: "Packages 1–23 are implemented… Package 10 was already satisfied by Chord's canonical structural diff implementation" (`:10-14`).
- 23 ordered packages (`pico-v5-handoff.md` headings): 1 records/cursors/memory tables · 2 memory document records · 3 SQLite · 4 JSONL publication · 5 JSONL reclamation · 6–7 tracker transaction core + typed access · 8 checkpoints/migration · 9 conversation document forks · 10 Chord structural array ops · 11 Chord document state · 12 document watches · 13 openable Harness · 14 durable task runtime · 15 first no-tool chat turn · 16 first coding-agent tool turn · 17 live UI/product state · 18 ownership + subagents · 19 structured concurrency · 20 compaction + overflow · 21 extensions + per-conversation agents · 22 lifecycle + final conformance · 23 task graph view. Each package section ends with the exact tests to write (e.g. Package 1: "Test reserved root identity and immutable creation, mixed atomic commits, rollback…").
- Commit rhythm: a `docs(durable): specify …` commit precedes each `feat(durable): … (Package N)` — 15: `034be02d2` → `837d8a16b`; 16: `0a7a4eca6` → `445770e03`; 17: `749e45ae6` → `47f65f1a3`; 18: `9f1013506` → `2532a0bef` (+ fix `1b347794e`); 19: `96377f5c2` → `03180653c`; 20: `b72cf98ec` → `ed0d6b91b`; 21: `4bae86775` → `b56702ad3`; 22–23: `5b5ccddfa` → `49683a364`.
- Design-research artifacts kept alongside: `docs/chord-delta-findings.md` (measured rejection of an ID-addressed graph tracker: ready heap 430 vs 139.5 MiB, import 1,119 vs 0.02 ms; weak caches 3,526 → 204 MiB retained but cold traversal 2.4 → 6.3 s; "No measured design … simultaneously achieved low retained and transient memory…", `:182-358`); `docs/pico-v5-chord-usage.md` (commit-line pipeline, "No visible-undurable path exists", `:353-366`); `packages/chord/PLANNING.md` (layering, forbidden vocabulary, open decisions `:839-842`).
- Who implements: handoff phrasing ("stop for user review") targets an implementing agent; commits are authored by humans (Mario Zechner 79, Armin Ronacher 28, Christian Klotz 6 on `packages/durable`) — agent involvement inferred from the handoff, not attested in trailers (unverified).

## Constants
| name | value | path:line |
|---|---|---|
| handoff packages | 23 | `pico-v5-handoff.md:14` |
| spec size | 4747 lines | `packages/durable/docs/spec.md` |

## Evolution
- 2026-05-03 `a5b27367d` first in-agent `AgentHarness` in `packages/agent/src/harness`; 2026-05-14 `b7ea82105`, `846906e4d`; 2026-06-10 `f0ccbbf01`/`9ab129267` ("Models is the harness's only auth path").
- 2026-08-16 → 2026-09-01: ad hoc remote runtime iteration (`33dac3623`, `f8a6e670d`, `353c990f4` mini, `1d0d110ab` Radius, Chord `34dc9d055`/`86bac52f9`/`1a7bc80e7`) — 9 protocol versions in a month.
- 2026-09-07 `73f3257dd` "preserve pico2 spike design for review" — start of spec-first redesign; 2026-09-09–13 `e045ed2f3` … `99a3948c4` design drafts, contracts; 2026-09-14 `46b66c59a` hardened pico3 kernel.
- 2026-09-15–17 `56cd5989e`, `c8e4a5a55`, `b02eef418`, `729d5cb74` replace obsolete Pico prototypes with Pico5 spec set (`729d5cb74` deletes 40.5k lines); 2026-09-17 `7e1950768` experimental micro agent (later removed).
- 2026-09-18 `080160162` Pico moved into `packages/durable`; 2026-09-22–28 storage/doc/task packages (`8158b0321`, `5901c9b9e`, `898ab8040`, `b313731b8`, `19a0361be`, `5d4de953c`, `507d7649e`, `35180b9df`, `3883fcb1f`, `cb7969d21`).
- 2026-09-29 → 2026-10-01 Packages 15–23 (above); 2026-10-01 `48dd1e2f0` client/server ported onto pi-durable, `7fd478a2e` old harness removed (105k lines), `a13d35a74` v1.0.0.
- 2026-10-05 pi-env created → released in one day (13 commits, `ba03e03f2` … `7c10bd433`) → [[pi--remote-execution-env]].

## Evidence commits
`73f3257dd` `729d5cb74` `080160162` `034be02d2` `837d8a16b` `0a7a4eca6` `445770e03` `749e45ae6` `47f65f1a3` `9f1013506` `2532a0bef` `1b347794e` `96377f5c2` `03180653c` `b72cf98ec` `ed0d6b91b` `4bae86775` `b56702ad3` `5b5ccddfa` `49683a364` `7fd478a2e`

## Quirks
- Footguns documented rather than enforced (long transactions holding the mutation line, unstable prompt text breaking caches, guard extensions omitted from array selection, JSONL without fsync) (`spec.md:4596-4731`).
- `pi.` name prefix reserved "by convention only. Nothing enforces it" (`spec.md:4695-4698`).
- Stable pi still runs the in-memory `packages/agent` loop; whether it migrates to pi-durable is not stated (unverified intent).

## Failures
[[rewrite-drops-test-coverage]] · [[shared-worktree-agents-clobber-each-other]]
