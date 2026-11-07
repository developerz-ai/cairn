# 🛠️ Implementation — how

Design-stage specs. Code follows these; when they disagree, fix the doc in the same PR.

| # | Doc | Hook |
|---|---|---|
| 01 | [Repo layout](01-repo-layout.md) | Modules, layering, `bin/` scripts |
| 02 | [Object formats](02-object-formats.md) | Canonical encoding, ids, chunks, freeze checklist |
| 03 | [Storage](03-storage.md) | MetaStore/BlobStore, schema, Yugabyte rules |
| 04 | [Protocol](04-protocol.md) | gRPC services, JSON gateway, `GET /o/<id>` |
| 05 | [Derived data](05-derived-data.md) | Framework, types, spine rewrite |
| 06 | [Landing engine](06-landing-engine.md) | Batch leader, state machine, limits |
| 07 | [Client and caches](07-client-and-cache.md) | `cairnd`, `cairn-cache`, checkout, status |
| 08 | [C++ standards](08-cpp-standards.md) | Toolchain, errors, ownership, style |
| 09 | [C++ performance](09-cpp-performance.md) | Threads, sync, I/O, algorithms, libs, benchmarks |
| 10 | [Memory](10-memory.md) | Arenas, pools, slabs, zero copy, containers |
| 11 | [Testing](11-testing.md) | DST, fuzz, property, differential |
| 12 | [Build and CI](12-build-and-ci.md) | Presets, deps, guards, CI jobs |
| 13 | [Web forge](13-web-ui.md) | Ultimate primitives mapping, identity, data |
| 14 | [Deploy and ops](14-deploy-and-ops.md) | Images, compose, observability, backups |
| 15 | [Roadmap](15-roadmap.md) | M0–M5 with gates, risks |
| 16 | [Virtual filesystem](16-virtual-filesystem.md) | FUSE, NFS/FSKit, ProjFS, overlay, prefetch |
| 17 | [Object storage](17-object-storage.md) | S3 layout, packing, presigned URLs, providers |
