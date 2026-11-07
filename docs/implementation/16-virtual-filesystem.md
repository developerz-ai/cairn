# 🪟 Virtual filesystem

> The repo looks fully checked out; bytes arrive on first read. Writes go to a local overlay. One core (`src/wc/vfs`) with a thin adapter per OS.

## Backends
| OS | Backend | Notes |
|---|---|---|
| Linux | **FUSE** (libfuse3, low-level API) | `FUSE_PASSTHROUGH` (kernel ≥ 6.9) for materialized files: reads bypass userspace. io_uring FUSE transport (`FUSE_IO_URING`, kernel ≥ 6.14) when available |
| macOS | **NFSv3 loopback** (EdenFS approach) default; **FSKit** (macOS 15.4+) via macFUSE 5 / FUSE-T optional | No kext required for NFS or FSKit |
| Windows | **ProjFS** (Projected File System) | Later; same core. Placeholders hydrate on read |
| Containers / CI | FUSE in a privileged sidecar, or no VFS (sparse / API only) | — |

## Core model
```
Inode table (u64 → node)       node = { tree_entry | overlay_file | overlay_dir, state }
state: virtual → hydrated(cached blob id) → modified(overlay)
```
- **Snapshot**: mounted at a changeset; root tree id pins the view. `cairn update` swaps the root atomically; unmodified hydrated files re-point, modified ones stay in the overlay.
- **Lookup**: `lookup(parent, name)` decodes the parent tree (cached, SoA layout, binary search) → no network if the tree is local.
- **Read**: first `open` → fetch blob via `cairnd` → materialize to the shared object store → passthrough fd. Large chunked blobs fetch only the chunks covering the read range (prefetch ahead).
- **Write**: copy-up on first write into `overlay/` (local disk); metadata changes (chmod, rename, delete) recorded in an overlay journal (append-only, fsync batched).
- **Status**: free. The overlay journal *is* the dirty set; no scanning, no watcher needed.
- **Commit**: hash overlay files (BLAKE3, parallel), upload missing, `CreateChangeset` with the delta.

## Performance
| Concern | Approach |
|---|---|
| `stat` storms (build tools) | Attributes from tree entries (size stored in tree entry for files); long kernel attr/entry TTLs, invalidate on `update` |
| `readdir` of huge dirs | Prolly-tree dirs, streamed readdir with offsets |
| Build tool cold start | Prefetch profiles: record paths read by a build, prefetch on next mount (EdenFS-style) |
| Thundering reads | Single-flight per blob id |
| Kernel round trips | FUSE writeback cache, `max_read` 1 MiB, `splice` read/write, multithreaded loop pinned per core |
| Large binaries | Chunk-range fetch; never hydrate a 4 GB asset to read its header |

## Safety
- Mount dies → files remain readable from object store via `cairn wc recover` (overlay is plain files + journal).
- Unmount refuses with dirty overlay unless `--keep-overlay` (default keeps it).
- Case-insensitive host FS (macOS/Windows): collisions detected at `lookup`, surfaced as an error file + `cairn status` warning.

## Phasing
1. M1: sparse checkout (real files) + stat cache + watcher.
2. M5: Linux FUSE + overlay + prefetch profiles.
3. M5+: macOS NFS/FSKit; Windows ProjFS.
