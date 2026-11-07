---
paths: ["src/object/**", "src/cache/**", "src/server/**", "src/land/**", "src/derive/**", "src/wc/**"]
---
# Hot-path rules

- **0 heap allocations per request** in steady state. Use the request arena (`ctx.arena()`), `pmr` containers, `small_vector`.
- Nothing allocated in an arena escapes it. Copy out explicitly.
- Views not copies: decode into `span`/`string_view` over the frame.
- Pass compressed frames through untouched; never recompress on a cache hit.
- No virtual calls / `std::function` in inner loops — templates or concepts.
- No locks across `co_await`, I/O or callbacks. Never block a gRPC reactor.
- Batch: N ids per RPC, one SQL pipeline, one io_uring submit.
- Bounded: every cache/queue has a byte cap + metric.
- Validate untrusted sizes **before** allocating.
- Perf claims need `bin/bench` before/after in the PR.
- Docs: `docs/implementation/09-cpp-performance.md`, `10-memory.md`.
