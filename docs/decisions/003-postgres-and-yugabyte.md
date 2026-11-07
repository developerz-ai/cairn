# 003 — Metadata on PostgreSQL (single node) and YugabyteDB YSQL (multi-node), one schema
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
Need transactions, range scans and a single-row CAS; must self-host on one box and scale out. Scion uses Spanner.

## Decision
`MetaStore` interface; `pg` and `yb` backends share one SQL schema via libpq. Blobs never in the DB.

## Alternatives considered
- **CockroachDB** — why not: license changes (2024) make self-hosting at scale costly; CarterT27/scion already covers it.
- **FoundationDB** — why not: great KV + DST story but no SQL for ops/debugging; kept as optional later backend.

## Consequences
- Easier: one-box install with stock Postgres; familiar ops.
- Harder: hot `main` ref row on a Raft store — needs batching.
- Revisit when: M3 benchmark shows YSQL cross-shard p99 too high for land rounds.
