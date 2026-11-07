# 🧱 Data model

> Three immutable object kinds hold repository state. Everything else is a derived, rebuildable index.

## Objects
Id = BLAKE3-256 over `type tag ‖ canonical bytes`. The tag keeps kinds from colliding.

| Object | Contents | Notes |
|---|---|---|
| **Blob** | File bytes, zstd at rest | < 1 MiB: one object. Larger: manifest of FastCDC chunks so edits dedupe |
| **Tree** | Sorted entries `(name, mode, child id)` | Large directories become prolly-style sharded trees (phase 2): one added file touches O(log n) nodes |
| **Changeset** | Parents, author, committer, message, timestamp, stable change id, metadata map (agent id, task id), sorted delta `(path, op, mode, blob id)` against first parent | Created only by the server. The delta is the unit of write |
| **Derived record** | Value keyed by `(kind, version, changeset id)` | Never part of a changeset id; rebuild or upgrade freely |

## Three choices
- **Delta against first parent only.** Trunk-based monorepo: history and blame matter on the main line. A merge's second parent is a plain pointer.
- **Canonical hand-written binary encoding** for hashed objects: varints, sorted fields, test vectors in the repo. Protobuf only on the wire (its encoding is not canonical).
- **Stable change id** (Jujutsu idea), separate from the content hash. Server-side rebases keep an identity agents and tools can follow.

## Root tree
Derived, computed **eagerly** on write: parent root + delta, rewriting only the path spine. Reads never wait for it.

## Refs
- Few: `main`, `release/*`. Each row holds `target`, `version`, and the changelog sequence.
- Every move is a versioned compare-and-swap, logged to `ref_log`.
- No per-agent branches on the server. An agent's unlanded work is a stack of changesets pointed to by its change ids.

## Invariants
| Invariant | Check |
|---|---|
| Object bytes hash to their id | Verified on every read, every tier |
| A changeset's delta applied to parent root = its root tree | Derivation test + `cairn fsck` |
| Ref versions strictly increase by 1 | DB constraint + DST invariant |
| `ref_log` replays to current refs | `cairn fsck` |
| Derived values are a pure function of `(changeset, parents' values)` | Recompute-and-compare property test |
| Redacted blobs keep their id | Redaction test |

## Derived types (first set)
| Type | Eager? | Used by |
|---|---|---|
| Root tree | yes | Checkout, everything |
| Path history | no | `log <path>` |
| Blame graph (per file version) | no | `blame` |
| Commit-graph segments + skip list | yes | `log`, `merge-base`, `is-ancestor` (O(merges)) |
| Copy/rename detection | no | `log --follow`, blame |
| Trigram index | no | `grep`, `find` |
| Git commit mapping | no | Read-only git mirror |

Formats and encoding detail: [implementation/02-object-formats.md](../implementation/02-object-formats.md).
