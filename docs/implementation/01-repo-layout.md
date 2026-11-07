# 📦 Repo layout (planned)

> One monorepo, best language per part: C++23 for core, TypeScript (Ultimate) for the forge. One set of `bin/` scripts, one CI.

```
cairn/
├── CLAUDE.md                 # agent/dev contract (AGENTS.md points here)
├── README.md
├── CMakeLists.txt  CMakePresets.json
├── cmake/                    # cairn_library(), cairn_test(), warnings, sanitizers, deps.cmake (CPM)
├── proto/cairn/v1/*.proto    # wire contract (gRPC); versioned, append-only
├── src/
│   ├── core/                 # error, assert, ids, arena, hash, varint, span utils, rng, clock
│   ├── object/               # canonical encode/decode: blob, tree, changeset, chunk manifest
│   ├── store/                # MetaStore + BlobStore interfaces; pg/, yb/, s3/, fs/, mem/, sqlite/
│   ├── derive/               # derived data framework + types (root tree, path hist, blame, segments)
│   ├── land/                 # land engine: rebase, merge, CAS, batch leader, idempotency
│   ├── merge/                # three-way merge, diff (Myers/histogram)
│   ├── net/                  # transport interface, gRPC adapter, HTTP gateway, sim transport
│   ├── server/               # cairn-server main + RPC handlers
│   ├── cache/                # cairn-cache (regional)
│   ├── daemon/               # cairnd: node object store, snapshot, LRU/pinning
│   ├── wc/                   # working copy: sparse, stat cache, watcher, FUSE overlay
│   ├── cli/                  # cairn CLI, --json schemas
│   └── sim/                  # deterministic simulation harness (DST)
├── tests/                    # unit/, integration/, dst/, fuzz/, golden/, vectors/
├── bench/                    # micro (nanobench) + macro (land storm, checkout)
├── tools/                    # importer (libgit2), fsck, replay/bisect, loadgen
├── apps/web/                 # cairn-web forge (Ultimate app)
├── docker/                   # Dockerfiles, docker-compose.yml, helm/
├── bin/                      # setup, dev, build, test, check, fmt, bench, fuzz, nightly
├── scripts/lint/             # guards: layering, raii, determinism, no-throw, contracts
└── docs/                     # idea/, implementation/, decisions/, plans/
```

## Layering (enforced)
```
core ← object ← store ← derive ← merge ← land ← server
                  ↑                         ↑
               net (adapters only)       cache, daemon, wc, cli
sim depends on everything; nothing depends on sim.
```
- CMake target deps + `scripts/lint/layering.sh`. A forbidden include fails `bin/check`.
- Heavy third-party headers (gRPC, libpq, aws) only in adapter `.cpp` files behind pimpl.

## `bin/` scripts
| Script | Does | Budget |
|---|---|---|
| `bin/setup` | Toolchain check, deps fetch, compose up Postgres + Garage (S3) | first run < 10 min |
| `bin/dev [server\|web\|all]` | Build + run with hot reload of web | < 5 s to serve |
| `bin/build [preset]` | CMake configure + build | 1-file rebuild + link < 5 s |
| `bin/test [filter]` | Unit tests, all cores | < 10 s |
| `bin/check` | fmt-check + lint + tidy + unit + integration + web `x verify`. **≡ CI** | < 5 min |
| `bin/fmt` | clang-format + biome | < 3 s |
| `bin/bench` | Benchmarks, compare to baseline JSON | — |
| `bin/fuzz <harness>` | libFuzzer run with corpus | — |
| `bin/nightly` | DST seeds sweep, TSan, long fuzz, soak | hours |

Build tooling detail: [12-build-and-ci.md](12-build-and-ci.md).
