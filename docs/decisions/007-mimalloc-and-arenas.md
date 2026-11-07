# 007 — mimalloc globally, arenas and slabs on hot paths
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
Hot paths must not hit the global heap; long-lived allocations need a fast, low-fragmentation allocator.

## Decision
mimalloc linked into every binary. Request/round arenas, per-thread pools and huge-page slabs per `docs/implementation/10-memory.md`.

## Alternatives considered
- **jemalloc** — why not: revived in 2026 (5.4) and strong; kept as A/B benchmark, not default.
- **tcmalloc** — why not: no releases, Bazel-centric, hard to package.

## Consequences
- Easier: predictable latency; alloc counts as a tested metric.
- Harder: arena lifetime bugs — mitigated by ASan poisoning and the "nothing escapes" rule.
- Revisit when: jemalloc wins the A/B on `cairn-cache` by > 10%.
