# 🗺️ Architecture overview

> A stateless server fleet in front of two stores, with three cache tiers between it and clients.

```
cairn CLI ─unix socket─▶ cairnd ─gRPC─▶ cairn-cache ─gRPC─▶ cairn-server ─▶ MetaStore (Postgres / YugabyteDB)
                         (node)          (regional)          (fleet)     └─▶ BlobStore (disk / S3)
                                                               ▲
            Web UI (Ultimate app) ─HTTP/JSON gateway───────────┘
            CI / verifier / bots ──gRPC (no clone)─────────────┘
```

## Components
| Component | Role | State |
|---|---|---|
| `cairn` | CLI: checkout, status, diff, commit, land, log, blame, grep. `--json` with versioned schemas | Working copy |
| `cairnd` | One per machine. Shared object store for every job on the node. LRU + pinning. Starts from a periodic snapshot | Bounded disk cache |
| `cairn-cache` | Regional. Streams objects from memory in the on-disk format `cairnd` consumes. No negotiation, no delta compression | Memory + disk, disposable |
| `cairn-server` | Authoritative. Object RPCs, landing, history, derived data. Stateless; scale by adding nodes | None (stores hold it) |
| `cairn-web` | Ultimate app: browse, history, blame, diff, lands, verifier, admin | Own Postgres database (UI state only) |
| MetaStore | Changesets, trees, refs, ref log, derived records, path history | Transactional |
| BlobStore | Blob and chunk bytes | Object storage |

## Tier rule
Each tier speaks the same object RPCs, but only calls the **next tier up**. A regional failure cannot stampede the servers.

## Object fetch (no negotiation)
Client never says what it has. It asks only for what it lacks:
1. Root tree of a changeset.
2. Missing child trees, batched, breadth-first, pruned by sparse scope.
3. Missing blobs, batched.

Every request is one immutable object: resumable, cacheable, CDN-friendly via `GET /o/<id>`.

## RPC groups
| Group | Examples |
|---|---|
| Objects | `GetTrees`, `GetBlobs`, `HasObjects`, `PutBlobs` |
| Write | `CreateChangeset`, `Rebase`, `Merge`, `Land` |
| History | `Log(path)`, `Blame`, `MergeBase`, `IsAncestor`, `Diff(a, b, paths)` |
| Refs | `ResolveRef`, `WatchChangelog` |
| Locks | `Lock`, `Unlock`, `WatchLocks` |
| Search | `FindFiles`, `Grep` |
| Admin | ACLs, principals, tokens, redaction |

## Working copy modes
| Mode | When | How |
|---|---|---|
| Sparse (v1) | Default | Materialize listed prefixes; stat cache + inotify/FSEvents |
| Virtual (v2) | Huge repos | FUSE (Linux), NFS loopback/FUSE (macOS); fetch on first read; writes to local overlay |
| None | CI, bots | Call the server directly |

Detail: [implementation/](../implementation/README.md).
