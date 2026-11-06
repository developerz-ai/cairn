# 🪨 Cairn

**Self-hostable, server-centric source control for monorepos worked on by swarms of agents.** C++23 client and server, plus a web UI built on [Ultimate](https://github.com/developerz-ai/ultimate).

> Status (2026-11): design stage, no code yet. "Cairn" is a working title.

## Why

Git assumes every clone holds the full history and every merge goes through a queue. With hundreds of agents landing changes, that breaks down. Cairn moves history and writes to the server:

- **Server owns history.** Commits, rebases and lands all run server-side. Clients are always online and fetch only what they need.
- **Land by compare-and-swap.** No merge queue. Concurrent lands are batched into one ref update per round, verified after the merge, and failures are bisected and reverted automatically.
- **Cost scales with the change.** A changeset is a delta against its first parent. Root trees, blame, path history and the commit graph are derived indexes.
- **Immutable, content-addressed objects** (BLAKE3, zstd). Any cache tier can serve them without coordination.
- **Optional, sparse working copies**: sparse checkout first, then a FUSE virtual filesystem. Jobs that only need a diff or blame call the API directly.

## Architecture

```
cairn CLI ─▶ cairnd (node daemon) ─▶ cairn-cache (regional) ─▶ cairn-server ─▶ MetaStore + BlobStore
                                                                  ▲
                                               web UI (Ultimate) ─┘
```

| Part | Role |
|---|---|
| `cairn-server` | Stateless fleet: object RPCs, landing, history, derived data |
| `cairnd` | One per machine; shared, bounded on-disk object cache |
| `cairn-cache` | Regional tier serving objects straight from memory |
| `cairn` | CLI: checkout, status, diff, commit, land, log, blame; `--json` everywhere |
| Web UI | Browse, history, blame, land activity, verifier status |
| Metadata | PostgreSQL (dev, small teams) or YugabyteDB (multi-node) |
| Blobs | Local disk or any S3-compatible store |

Protocol: gRPC with protobuf on the wire. Hashed objects use a canonical hand-written encoding with checked-in test vectors.

## Self-hosting

Single node: `cairn-server` with PostgreSQL and local blobs. Scale out by adding server nodes, switching to YugabyteDB and S3, and placing caches near clients. No external services are required.

## Engineering

C++23 (Clang 18+, GCC 14+), CMake + Ninja, `std::expected` errors, no exceptions across module boundaries. Hot paths borrow from game engines (as in our railroads engine): data-oriented layouts, arena allocation, zero-copy object streaming, io_uring disk I/O and deterministic simulation testing. Every decoder is fuzzed, and ASan, UBSan and TSan run on every change.

## Roadmap

| Milestone | Scope |
|---|---|
| M0 | Object formats, test vectors, in-memory stores |
| M1 | Single-node server on PostgreSQL, CAS landing, sparse checkout, CLI |
| M2 | Derived data: path history, blame, segmented commit graph; git import |
| M3 | YugabyteDB, S3, `cairnd` and `cairn-cache` tiers, land batching |
| M4 | Path ACLs, service principals, redaction, verifier with auto-revert |
| M5 | Virtual working copy, git protocol v2 read path, read-only git mirror |

Not planned for v1: offline work, two-way git sync, built-in code review, search or CI (Cairn exposes APIs for them).

## Credits

Cairn builds on published work from Meta's **Mononoke** and Bonsai data model, **Sapling**'s segmented changelog, **EdenFS**, Google's **Piper** and CitC, **Jujutsu**, **Dolt**'s prolly trees, and Anthropic's description of **Scion**. It is independent and not affiliated with any of them.
