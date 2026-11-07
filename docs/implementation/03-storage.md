# 🗄️ Storage

> Two narrow interfaces keep backends swappable: `MetaStore` (transactional) and `BlobStore` (bytes). One SQL schema serves PostgreSQL and YugabyteDB YSQL.

## Interfaces
```cpp
struct BlobStore {
  virtual Task<expected<void, Error>> put(ObjectId, std::span<const std::byte> frame) = 0;
  virtual Task<expected<Frame, Error>> get(ObjectId) = 0;
  virtual Task<expected<Bitset, Error>> exists(std::span<const ObjectId>) = 0;
  virtual Task<expected<std::vector<Frame>, Error>> get_batch(std::span<const ObjectId>) = 0;
};
struct MetaStore {
  virtual Task<expected<Txn, Error>> begin(Isolation) = 0;     // Txn: get/put/scan/commit
  virtual Task<expected<RefCas, Error>> cas_ref(RefName, Version expected, ChangesetId next) = 0;
};
```
| Backend | Use |
|---|---|
| `mem` | Unit + DST (deterministic) |
| `sqlite` | Single-binary local dev, tests |
| `pg` (libpq, pipeline mode) | Dev, small teams, single node |
| `yb` (libpq to YSQL) | Multi-node production |
| `fs` blobs | Single node; sharded dirs `ab/cd/<id>` |
| `s3` blobs | Any S3-compatible (AWS, R2, GCS, Garage, SeaweedFS, Ceph RGW, B2). Detail: [17-object-storage.md](17-object-storage.md) |
| `fdb` | Optional later; pure KV layout |

## Schema (sketch)
```sql
CREATE TABLE changesets (id BYTEA PRIMARY KEY, parents BYTEA[], body BYTEA, change_id BYTEA, ts TIMESTAMPTZ);
CREATE TABLE trees      (id BYTEA PRIMARY KEY, body BYTEA);      -- small inline; large → blob store
CREATE TABLE refs       (name TEXT PRIMARY KEY, target BYTEA, version BIGINT, seq BIGINT);
CREATE TABLE ref_log    (ref TEXT, version BIGINT, target BYTEA, actor TEXT, op TEXT, ts TIMESTAMPTZ,
                         PRIMARY KEY (ref, version));
CREATE TABLE derived    (kind TEXT, ver INT, cs BYTEA, value BYTEA, PRIMARY KEY (kind, ver, cs));
CREATE TABLE path_hist  (path TEXT, seq BIGINT, cs BYTEA, PRIMARY KEY (path, seq));
CREATE TABLE land_idem  (key TEXT PRIMARY KEY, result BYTEA, expires TIMESTAMPTZ);
CREATE TABLE locks      (path TEXT PRIMARY KEY, ref TEXT, holder TEXT, change_id BYTEA, ts TIMESTAMPTZ);
CREATE TABLE packs      (id BYTEA PRIMARY KEY, sealed BOOL, size BIGINT);
CREATE TABLE pack_index (obj BYTEA PRIMARY KEY, pack BYTEA, off BIGINT, len INT);
CREATE TABLE git_map    (git_sha BYTEA PRIMARY KEY, cs BYTEA);
```
Migrations: forward-only SQL files, applied by `cairn-server migrate`, checked in CI against both engines.

## YugabyteDB rules
- **Hash-shard** by object id (`changesets`, `trees`, `derived`): writes spread across tablets.
- **Range-shard** `path_hist (path ASC, seq ASC)`: a file's history is one scan.
- **Hot row**: `refs['main']`. Batch leader → one CAS per round. Ref row carries `seq`; no global counter.
- **Short transactions**: changeset + delta rows + ref in one txn. Derivation after commit.
- **No blobs in the DB.** Rows hold ids and small metadata.
- Benchmark cross-shard txn p99 on the target topology before M3 gate.

## PostgreSQL rules
- `pgbouncer`/pgcat in transaction mode; prepared statements via protocol-level names.
- libpq **pipeline mode** for batch reads (`GetTrees` misses): one round trip for N lookups.
- `COPY` for bulk import.

## Read path
```
in-process LRU (hot trees, refs) → regional cache → MetaStore/BlobStore
```
Database sees mostly misses and writes. Scion reports > 99% hit per tier.

## GC
Objects are immutable; nothing is deleted in normal operation. Unreachable uploads (blobs uploaded for a land that never happened) are swept after a grace period by a mark phase over refs + pending stacks. Redaction replaces bytes with a tombstone; id unchanged.
