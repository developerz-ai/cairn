---
paths: ["src/store/**", "migrations/**"]
---
# Storage rules

- One SQL schema for PostgreSQL and YugabyteDB YSQL. Test migrations on both.
- Migrations forward-only. Never edit an applied migration.
- Transactions short: changeset + delta + ref in one txn; derivation after commit.
- No blobs in the database. Rows hold ids + small metadata.
- YSQL: hash-shard by object id; range-shard `path_hist (path, seq)`.
- libpq pipeline mode for batched reads. No ORM.
- S3: conditional writes for idempotent puts, multipart over threshold, pack small objects. See `docs/implementation/17-object-storage.md`.
- Doc: `docs/implementation/03-storage.md`.
