# 🔁 Git interop

> Teams arrive from git and some tools need git forever. Interop is one-way plus read-only, in three steps.

| Step | Milestone | What |
|---|---|---|
| **1. Import** | M2 | libgit2 replays git history into changesets; records `git sha ↔ changeset id` mapping. One way |
| **2. Read-only mirror** | M5 | Derived type maps changesets to git commits, so GitHub/GitLab can mirror `main` for tools that need git |
| **3. Git protocol v2 façade** | M5+ | `ls-refs` + `fetch` with `filter` served from Cairn objects. Stock git clients do sparse reads |

## Not doing
**Two-way sync.** It forces git's data model on Cairn's write path (SHA-1/SHA-256 commit ids, client-side merges, arbitrary branches).

## Import details
- Linearizes first-parent history; merges keep their second parent as a pointer.
- Blob ids are recomputed (BLAKE3); git blob shas kept in the mapping for lookup.
- Resumable and idempotent: re-running continues from the last mapped commit.
- Gate (M2): a 100k-commit repo imports and answers `blame` without a local clone.
