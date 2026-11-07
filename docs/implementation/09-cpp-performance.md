# ⚡ C++ performance

> Cost scales with the change. Data-oriented layouts, shared-nothing cores, io_uring, zero copy, no allocations on hot paths, and every claim backed by before/after numbers.

Versions as of 2026-11. Memory is its own doc: [10-memory.md](10-memory.md).

## Toolchain for speed
| Lever | Setting |
|---|---|
| Compiler | Clang 23 (release), GCC 16 (CI). Both build everything |
| Target | `-march=x86-64-v3` (AVX2) default; `x86-64-v2` fallback package; `armv8.2-a` for arm64 |
| LTO | ThinLTO in `release` |
| PGO | Instrumented profile from `bench` macro workloads → `-fprofile-use`; AutoFDO later |
| Post-link | BOLT on `cairn-server` and `cairn-cache` (5–15% typical on server code; measure) |
| Frame pointers | `-fno-omit-frame-pointer` in all builds: always-on profiling beats ~1% |
| Linker | lld; mold v3 locally |
| Stdlib hardening | libc++ `fast` mode in release (bounds checks, cheap) |

C++26 status: reflection and contracts only in GCC 16 (`-freflection`). Don't depend on them; gate behind CI matrix if tried.

## Threading model
| Process | Model |
|---|---|
| `cairn-server` | gRPC callback reactors on a fixed pool (≈ cores); CPU work (hash, compress, merge, derive) on a work-stealing pool; **never block in a reactor** |
| `cairn-cache` | **Thread-per-core, shared-nothing**: id-hash → owning core; one io_uring ring per core; no locks on the data path |
| `cairnd` | Small pool + one io_uring ring; Unix-socket server |
| `cairn` CLI | Work-stealing pool for hash/decompress/write; io_uring batched file writes |
| Land leader | One coroutine per ref; single writer; batching is the concurrency |

- Async = C++20 coroutines (`Task<T>`) over our executor. Senders/receivers via **stdexec** if it pays off; std `<execution>` not yet in libc++/libstdc++ releases.
- CPU pool: oneTBB or Taskflow; or own Chase-Lev deques. Coarse jobs (≥ 50 µs), private outputs, ordered merge (railroads `parallel_map_ordered`) so results are deterministic at 1 or N workers.
- No `std::thread` raw; `std::jthread` owned by a pool.

## Synchronization
| Need | Use |
|---|---|
| Hot read-mostly state (ref tips, ACL snapshot, config) | **RCU / epoch**: readers load an immutable snapshot pointer (`std::atomic<std::shared_ptr>` or liburcu QSBR); writer publishes a new one |
| Concurrent cache map | `boost::concurrent_flat_map` or sharded maps (shard per core / 64 shards) |
| Queues | SPSC ring (rigtorp-style) between fixed pairs; MPSC for per-ref land queue; moodycamel for MPMC |
| Counters | Per-core `alignas(64)` counters, summed on read |
| Single-flight | Sharded in-flight map → shared future |
| Locks | Only off hot paths; never across `co_await` or I/O |

## I/O
- **io_uring** (liburing 2.15): registered files + buffers, multishot accept/recv, `SEND_ZC` for large frames, linked ops for write → fsync → rename.
- Disk cache: append-only segment files, `O_DIRECT` for big sequential reads, page cache for small hot objects.
- Plain HTTPS `GET /o/<id>`: `sendfile`/`splice` from segment files + kTLS. CDN-friendly, near-zero CPU.
- gRPC path: frames as `grpc::Slice` over cache memory (no copy).
- Batching everywhere: one RPC for N ids, one SQL pipeline for N lookups, one io_uring submit for N writes.

## Algorithms
| Problem | Algorithm |
|---|---|
| Ancestry, merge-base | Segmented changelog (O(merges)) + skip list |
| Changed paths since base | Sorted-merge of path-sorted deltas; Bloom/xor prefilter per segment |
| Tree diff | Walk two sorted trees in lockstep; skip equal subtree ids (O(changed)) |
| Text diff | Histogram diff (git-compatible output); Myers fallback; line hashing with rapidhash |
| Three-way merge | diff3 over histogram hunks; byte-level for binary = conflict |
| Large blob dedupe | FastCDC (own ~200-line impl, gear hash, normalized chunking) |
| Big directories | Prolly tree (content-defined node boundaries) |
| `HasObjects` | xor/binary-fuse filter snapshot, then exact lookup |
| Grep | Trigram index, roaring bitmap postings, intersect then verify with SIMD `memmem` |
| Cache eviction | S3-FIFO or CLOCK-Pro (scan-resistant, lock-free friendly) |
| Ref → leader | Rendezvous hashing |
| Blame | Incremental per file version from parent's blame + diff |

## CPU tricks (from game engines)
- **SoA + contiguous**: tree entries as parallel arrays; scans stay in cache, vectorize.
- **Hot/cold split**: hot fields (id, mode, name offset) separate from cold (metadata).
- **Branchless** compares in sorted merges; `std::countr_zero` over match masks.
- **Prefetch** next probe in batch lookups (`__builtin_prefetch`) — batch of N ids looked up interleaved.
- **SIMD**: BLAKE3 and zstd bring their own; for our loops use Highway or auto-vectorizable loops (no virtual calls, no `std::function` inside).
- **Static dispatch**: concepts/templates on hot paths, `virtual` only at adapter seams.
- **Fixed-tick background work**: GC, compaction, backfill run in budgeted slices `(tick + shard) % N`, deterministic under DST.
- **Dirty flags + derived caches** keyed on every input; property test cached == fresh.
- **Hash once**: BLAKE3 id computed while streaming the upload; never re-read to hash.

## Libraries (pinned via CPM)
| Area | Choice | Why |
|---|---|---|
| Hash map | boost.unordered 1.92 (`flat`, `concurrent_flat`), ankerl::unordered_dense 5.x | Fastest open addressing, concurrent variant |
| Fast non-crypto hash | rapidhash v3 (wyhash successor), xxh3 | Hash tables, line hashing |
| Crypto hash | BLAKE3 1.8 (C, SIMD dispatch, optional TBB) | Object ids |
| Compression | zstd 1.5.7 (dicts, `--long`, `--patch-from`) | Objects, packs, binary deltas |
| Filters/bitmaps | xor_singleheader 2.x, CRoaring 5.x | `HasObjects`, trigram |
| Concurrency | stdexec, oneTBB 2023.x / Taskflow 4, moodycamel, liburcu 0.15 | Pools, queues, RCU |
| I/O | liburing 2.15 | Disk + net |
| RPC | gRPC 1.85 (callback API), protobuf 36 with arenas | Wire |
| HTTP | nghttp2 (gateway, `/o/`) | Lightweight HTTP/2 |
| JSON | glaze 9 (write), simdjson 5 (read) | CLI `--json`, gateway |
| Allocator | mimalloc 3.x (jemalloc 5.4 as A/B) | See memory doc |
| Logging | quill 13 (async, ns-latency) | Hot-path-safe logs |
| Telemetry | opentelemetry-cpp 1.30 | Traces + metrics |
| Profiling | Tracy (zones in debug/bench builds), perf, bpftrace, uftrace, heaptrack | Find the hot spot |
| Crypto | AWS-LC (pairs with aws-sdk-cpp) or libsodium | Ed25519, TLS |
| DB | libpq (pipeline mode) | No ORM on hot path |

## Benchmarks (gates)
| Bench | Target |
|---|---|
| Encode/decode tree, 1k entries | < 20 µs |
| BLAKE3 throughput | ≥ 3 GB/s/core (AVX2) |
| `GetBlobs` from warm cache | ≥ 1 GB/s per server core, p99 < 2 ms |
| Warm sparse checkout, 1,500 files | ≤ 0.2 s |
| Lands/min, 50 agents, 16 cores | ≥ 1,500 |
| Allocations per object RPC | 0 (steady state) |

Every perf PR: before/after numbers from `bin/bench` in the description. No number, no merge.
