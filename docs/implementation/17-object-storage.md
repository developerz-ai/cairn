# ☁️ Object storage (S3 and friends)

> Any S3-compatible store holds the bytes. History and CAS stay in the MetaStore; S3 only stores immutable, content-addressed data.

As of 2026-11.

## Rules
1. **S3 is never the source of truth for refs.** CAS landing lives in Postgres/YugabyteDB.
2. **Writes are idempotent by construction**: key = object id. Overwrite with same bytes is harmless; `If-None-Match: *` used where supported to skip the upload.
3. **Pack small objects.** Per-request cost dominates for tiny objects. Objects < 256 KiB go into pack files (8–64 MiB); the MetaStore maps id → (pack, offset, length). Large chunks are single objects.
4. **Verify on read.** Every byte from S3 is hashed before use.
5. **Bytes bypass the server where possible**: presigned URLs for big uploads/downloads.

## Layout
```
<bucket>/<prefix>/
  o/ab/cd/<id>            # large objects / chunks (≥ 256 KiB)
  p/<pack-id>.pack        # small-object packs (immutable once sealed)
  p/<pack-id>.idx         # id → offset index (also in MetaStore)
  t/<id>                  # tombstones for purged objects
```
Hex-prefix sharding spreads load across S3 partitions.

## Write path
| Size | Path |
|---|---|
| < 256 KiB | Server buffers into an open pack (per node); sealed at 64 MiB or 2 s; MetaStore index rows written in the same land txn as references |
| 256 KiB – 64 MiB | Single `PutObject` with `If-None-Match: *` |
| > 64 MiB (packs only; chunks are smaller) | Multipart, 16 MiB parts, parallel |
| Client upload of large files | `PutBlobs` returns presigned PUT URLs for missing chunks; client uploads directly; server verifies hash on first read |

Unsealed pack data is durable via a write-ahead copy in the MetaStore (small rows) until sealed.

## Read path
- `cairn-cache` and server LRU serve > 99%; S3 sees misses only.
- Pack reads use HTTP `Range` requests; adjacent objects coalesced into one range.
- Large downloads: presigned GET URLs handed to clients (optional; disabled when ACL redaction must be enforced per byte).
- Retries: exponential backoff with jitter on 503 SlowDown; per-prefix concurrency limits.

## Provider notes
| Provider | Conditional put | Notes |
|---|---|---|
| AWS S3 | ✅ `If-None-Match` (Aug 2024), bucket-policy enforcement (Nov 2024) | Strong read-after-write since Dec 2020. Max object 50 TB (Dec 2025) |
| Cloudflare R2 | ✅ `If-Match` / `If-None-Match` | No egress fees; good default for self-hosters |
| Google GCS (XML API) | Use `x-goog-if-generation-match: 0` | Preconditions rejected on XML multipart |
| Backblaze B2 | ❌ reported rejected (2025) | Works; skip the conditional header |
| Garage, SeaweedFS, Ceph RGW | Verify per version | Self-hosted options; **Garage is the dev default** |
| MinIO | — | Community edition in maintenance mode since Dec 2025, repo archived 2026. Not our default |

Capability detection at boot: probe conditional put on a scratch key; store result; never depend on it for correctness.

## Lifecycle and cost
- Hot packs/chunks: standard class. Archived (`keep_last`, archive command): lifecycle to infrequent/cold class; rehydrate on read with a clear "warming" status.
- Purge: delete bytes, write tombstone; id and size stay in history.
- GC: unreferenced uploads (abandoned lands) swept after 7 days by mark over refs + pending stacks; packs compacted when > 30% dead.

## Local disk backend
Same layout on a filesystem (`file:///data/blobs`): packs + large objects, `O_DIRECT` reads, io_uring. Default for one-box installs.

## Testing
Integration tests run against **Garage** (container) and the in-memory backend; nightly against R2 and AWS with a test bucket. DST simulates S3 latency, 503s and partial multipart failures.
