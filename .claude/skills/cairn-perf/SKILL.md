---
name: cairn-perf
description: Patterns for fast, allocation-free C++ in Cairn — arenas, SoA, batching, zero copy, io_uring, concurrency. Use when writing or optimizing hot-path code.
---
# Performance patterns

| Situation | Pattern |
|---|---|
| Per-request temporaries | `ctx.arena()` + `std::pmr` containers; nothing escapes |
| Fixed-size churn | per-thread pool / free list |
| Cached object bytes | slab + refcounted slice; serve frame as-is |
| Decoding | `span`/`string_view` views into the buffer, no per-entry strings |
| Scanning entries | SoA arrays; branchless sorted-merge |
| N lookups | one batch RPC / SQL pipeline / io_uring submit; prefetch next probe |
| Concurrent misses | single-flight by id |
| Read-mostly global state | RCU snapshot pointer |
| Counters | per-core, `alignas(64)` |
| CPU work | coarse jobs ≥ 50 µs on the pool, ordered merge |
| Hash map | `boost::unordered_flat_map` / `ankerl::unordered_dense` |

Measure: `bin/bench`, `perf record -g`, Tracy, `CAIRN_ALLOC_HOOK=1`. Docs: `docs/implementation/09-cpp-performance.md`, `10-memory.md`.
