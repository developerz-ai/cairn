# 🧠 Memory

> Zero allocations per request on steady-state hot paths. Arenas per request, slabs for the object cache, pools for fixed-size nodes, and the global allocator only for long-lived things.

## Rules
1. **Hot paths allocate from an arena**, never the global heap. Hot = object RPCs, tree walk, delta replay, land round, encode/decode.
2. **Lifetime decides the allocator** (table below). Pick by lifetime, not by size.
3. **Measure**: alloc counting hook in tests; `allocs_per_request` metric in `bench`. A regression fails the bench gate.
4. **Bounded everything**: every cache, queue and arena has a byte cap and a metric.
5. **Untrusted sizes never drive allocation** before validation (declared length ≤ limit, zstd frame content size ≤ cap).

## Allocator by lifetime
| Lifetime | Allocator | Example |
|---|---|---|
| One request / RPC | **Request arena** (monotonic, rewound at end) | Decoded trees, path vectors, reply builders |
| One land round | **Round arena** | Rebased deltas, merge buffers |
| One derivation | Derivation arena | Spine rewrite scratch |
| Fixed-size, high churn | **Pool / free list** (per thread) | Tree entry nodes, in-flight map entries, coroutine frames |
| Cache contents | **Slab** over huge pages, size classes | Compressed object frames in `cairn-cache` and server LRU |
| Process lifetime | Global (mimalloc) | Config, connection pools, indexes |
| Large file I/O | Registered io_uring buffers | Blob reads/writes, socket send |

## Request arena
```cpp
class Arena {                         // bump pointer over chained blocks
  std::byte* cur_; std::byte* end_; Block* head_;
public:
  void* alloc(std::size_t n, std::size_t align);   // inline fast path: align, bump, check
  void  rewind(Mark) noexcept;                       // O(1); keeps the first block hot
};
using ArenaResource = std::pmr::monotonic_buffer_resource;   // std containers: pmr::vector<T>
```
- First block lives in the coroutine/request frame (4–16 KiB inline): small requests never touch the heap.
- Blocks recycled through a per-thread block cache, not freed.
- **Rule**: nothing allocated in an arena outlives it. Results that escape are copied out explicitly (`arena_to_owned()`).
- Debug build: poison rewound memory (`0xDD`), ASan manual poisoning (`ASAN_POISON_MEMORY_REGION`).

## Object frames: zero copy
- Cache stores the compressed frame once in a slab; RPC replies reference it with a refcounted slice (`grpc::Slice` from external memory with a release callback). No copy from cache to socket.
- Decode works on `std::span<const std::byte>`; tree entries are `string_view`s into the decompressed buffer (flyweight). No per-entry string allocation.

## Data layout
- **SoA** for scans: tree entries as parallel arrays (`names` blob + offsets, `modes`, `ids`) for sorted-merge and binary search.
- **Small-buffer** containers for typical sizes: `small_vector<T, 8>` for parents, path components.
- **Interning**: path components interned per request (arena-backed hash set) so comparisons are pointer compares.
- Pack hot fields in ≤ 64 B; `alignas(std::hardware_destructive_interference_size)` on per-thread counters and queue heads (no false sharing).
- Prefer indices (`uint32_t`) over pointers inside arenas: half the size, relocatable.

## Global allocator
- **mimalloc** linked in all binaries (override `malloc`/`new`). Secure mode off in release; on in fuzz builds.
- Transparent huge pages `madvise` for slab regions; explicit 2 MiB pages via `MAP_HUGETLB` when available.
- NUMA: `cairn-cache` pins one shard per core and allocates its slab on that core's node.

## Containers
| Need | Use |
|---|---|
| Hash map, hot | `boost::unordered_flat_map` or `ankerl::unordered_dense` (open addressing, SIMD probing) |
| Map with stable refs | `boost::unordered_node_map` |
| Sorted, small | sorted `std::vector` + binary search |
| Ordered, big | B-tree (`absl::btree_map`) |
| Fixed capacity | `std::inplace_vector` (C++26) or own `StaticVector<T, N>` |
| Id-addressed graph | slot map + generation (`Id<Tag>`), as in railroads |
| Set membership at scale | Ribbon/xor filter (`HasObjects` prefilter), roaring bitmaps (trigram postings) |

## Memory budgets (initial)
| Process | Budget |
|---|---|
| `cairn-server` per request (p99) | ≤ 256 KiB arena, 0 global allocs |
| `cairn-cache` | Configured slab size; < 2% metadata overhead |
| `cairnd` | Configured disk cap; ≤ 256 MiB RSS |
| `cairn` CLI checkout 1,500 files | ≤ 128 MiB RSS |

## Tooling
ASan + LSan (CI), Valgrind for uninitialized reads, heaptrack for allocation flame graphs, `CAIRN_ALLOC_HOOK=1` counting `operator new` that reports call sites in tests.
