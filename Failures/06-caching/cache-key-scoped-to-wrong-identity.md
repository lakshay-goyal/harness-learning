---
type: failure
concepts: [session-affinity-cache-routing, cache-stable-prompt-prefix]
harnesses: [codex]
---
**Symptom** — Requests that share a prompt prefix landed on different cache shards:
- `prompt_cache_key` defaulted to the *thread* id, so a root agent and its subagents (different thread ids, same prefix) used different keys (`4aa950d456` 2026-07-14).
- Ephemeral forks lost the parent's cache affinity, because the ChatGPT backend routes by the `session-id` *header*, not only by the body key (`bc5957eac9` 2026-09-11).
- Guardian reviewer sessions needed their own stable key (`4ce563a873` / `3307240195` 2026-05-28).

**Root cause** — The cache key was chosen at the granularity of the conversation object (thread), not of the shared prefix. The backend also had two routing inputs (body key and header) that had to agree.

**Fix · [[codex]]**
- `4aa950d456` 2026-07-14 (#33035), "Use session IDs for prompt cache keys": root and subagent requests share the session id as key.
- `bc5957eac9` 2026-09-11 (#44862), "Preserve parent cache affinity for ephemeral forks".
- Current resolution (`codex-rs/core/src/client.rs:581-603`):
  - an explicit override;
  - else `"{source}:{parent_thread_id}"` for internal sessions;
  - else the session id.
- The `session-id` header carries the cache key for root agents.

**Lesson** — Choose the cache key at the granularity that shares the prefix (session or agent family), not the conversation object, and keep every routing input the backend uses (body key and headers) consistent. pi's opposite rule, a fresh key for side requests ([[side-request-cache-pollution]]), applies when the side request does *not* share the prefix.

Related: [[session-affinity-cache-routing]] · [[cache-stable-prompt-prefix]] · [[cache-affinity-lost-across-reopen]] · [[side-request-cache-pollution]] · [[codex--session-affinity-cache-routing|codex]]
