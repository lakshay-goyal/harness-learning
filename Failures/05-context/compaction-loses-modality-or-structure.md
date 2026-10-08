---
type: failure
concepts: [auto-compaction, image-normalization, compaction-cut-point]
harnesses: [codex]
---
**Symptom** — Local compaction flattened retained user messages into one text part, losing content boundaries/annotations and images (`[image, "what is this?"]` became text); remote compaction's retained budget ignored image cost, so image-heavy history retained more than budgeted.

**Root cause** — Keep-set code was written for text: rebuild-as-text for every kept message, token budget counted only text.

**Fix · [[codex]]**
- `6677fd827d` 2026-08-23 "Budget retained images during remote compaction (#40280)": "Remote compaction's retained-message budget counted text but not images, so image-heavy history could retain more context than the budget represented." — images charged at their estimate, kept atomic with their label tags, no older backfill after a boundary image does not fit (`codex-rs/core/src/compact_remote_v2_images.rs:24-98`).
- `4bd5b9fd09` 2026-08-04 keep image-resize notices attached during remote compaction (`codex-rs/core/src/compact_remote_history.rs:39-66`).
- `bd3d4d1436` 2026-09-25 (#48115) keep original parts for text-only messages that fit (`codex-rs/core/src/compact.rs:707-728`). The boundary message that is truncated still becomes text-only with media dropped (`:686-764`).

**Lesson** — Compaction keep-sets must budget and preserve every modality and content boundary, not just text.

Related: [[auto-compaction]] · [[image-normalization]] · [[compaction-cut-point]] · [[codex--compaction-cut-point|codex]]
