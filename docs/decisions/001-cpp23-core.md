# 001 — Core (server, caches, daemon, CLI) is written in C++23
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
Cache tiers serve compressed objects from memory at line rate; the server parses untrusted input and runs CPU-heavy merges, diffs and hashing. Existing open Scion-style projects are Rust or Go. We want control over memory layout, allocation and I/O scheduling, and we have in-house C++23 experience (railroads).

## Decision
C++23 with Clang (release) and GCC (CI). Errors as values, no exceptions, arenas on hot paths. Safety via fuzzing, sanitizers, hardened libc++ and a small trusted core.

## Alternatives considered
- **Rust** — why not: strongest safety story, but CarterT27/scion, openscion already occupy it; we want a C++ implementation with game-engine memory techniques.
- **Go** — why not: GC pauses and limited layout control on the cache tier.

## Consequences
- Easier: zero-copy serving, custom allocators, io_uring, direct use of BLAKE3/zstd/libpq/FUSE C APIs.
- Harder: memory safety is on us; fuzz + sanitizers are mandatory from M0.
- Revisit when: a memory-safety CVE ships in a decoder despite fuzzing.
