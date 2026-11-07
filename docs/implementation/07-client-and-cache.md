# 💾 Client, daemon and cache tiers

> Strict chain: CLI → `cairnd` → `cairn-cache` → `cairn-server`. Each tier serves the same object RPCs and only calls the next tier up.

## `cairn-cache` (regional)
- Stateless; scale by adding instances behind consistent hashing on object id.
- Memory tier: sharded, CLOCK-Pro/S3-FIFO eviction over compressed frames in large slabs (huge pages). Disk tier: append-only segment files + in-memory index; io_uring reads.
- Serves frames as-is (no recompress, no re-encode): `sendmsg`/zero-copy from slab to socket buffer.
- Request coalescing: concurrent misses for one id → one upstream fetch.
- Thread-per-core, shared-nothing shards; a request is routed to the owning core by id hash.

## `cairnd` (node daemon)
- One per machine; all jobs share its object store (`~/.cache/cairn` or `/var/cache/cairn`).
- Content-addressed files in a sharded dir, or a pack of segments + index (LMDB) for small objects.
- Bounded size, LRU + **pinning** (objects for active checkouts can't be evicted).
- Boots from a periodic snapshot (pre-warmed object pack for `main`), then fetches deltas.
- Serves CLI over a Unix socket; verifies every object id on write.

## CLI checkout algorithm
1. `ResolveRef(main)` → changeset → root tree.
2. BFS over trees within scope (path prefixes); batch `GetTrees` for missing ids.
3. Batch `GetBlobs` for missing files; decompress + verify on a worker pool.
4. Write files: `openat` in parallel per directory, write to temp, `renameat2` atomic; set mode; batch with io_uring.
5. Record stat cache (mtime, size, inode, id) for `status`.

## Status and commit
- `status`: watcher (inotify/fanotify, FSEvents) marks dirty paths; stat cache filters the rest; hash only candidates.
- `commit`: hash dirty files (BLAKE3 multi-threaded for large files), `HasObjects`, `PutBlobs` missing, `CreateChangeset` with the delta. Server builds the changeset.

## Virtual working copy (M5)
- libfuse3, low-level API, multithreaded loop; `FUSE_PASSTHROUGH` for materialized files where the kernel supports it.
- Inode table → tree entry; content fetched on first `open`/`read` through `cairnd`.
- Writes go to a local overlay (copy-up). `commit` snapshots the overlay.
- macOS: NFSv3 loopback server (EdenFS approach) or macFUSE/FUSE-T.
