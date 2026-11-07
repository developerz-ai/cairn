# 🔧 Build and CI

> CMake presets + Ninja + ccache, deps pinned with CPM. `bin/check` is the CI gate, locally.

## Presets
| Preset | Flags | Use |
|---|---|---|
| `dev` | `-O1 -g`, asserts on | Daily |
| `debug` | `-O0 -g`, libc++ extensive hardening | Debugger |
| `asan` | ASan + UBSan + LSan | CI, local |
| `tsan` | TSan | Nightly, concurrency changes |
| `fuzz` | libFuzzer + ASan | Fuzzing |
| `ci` | `dev` + `-Werror` + unity | CI gate |
| `release` | `-O3 -march=x86-64-v3`, ThinLTO, hardening fast | Shipping |
| `pgo` | release + `-fprofile-use` (+ BOLT post-link) | Shipping (M3+) |
| `bench` | release + frame pointers | Benchmarks, profiling |
| `coverage` | `--coverage` | Weekly |

## Dependencies (CPM, exact tags, static)
| Area | Lib |
|---|---|
| Hash | BLAKE3 (official C) |
| Compression | zstd |
| RPC | gRPC C++ + protobuf (adapter targets only) |
| DB | libpq |
| S3 | aws-sdk-cpp (S3 only) or SigV4 over libcurl |
| Allocator | mimalloc |
| Containers | boost.unordered (header-only subset), roaring |
| I/O | liburing, libfuse3 |
| Git | libgit2 (importer tool only) |
| Crypto | libsodium or BoringSSL (Ed25519) |
| JSON | glaze (CLI `--json`, config) |
| Logging/telemetry | quill (async logging), opentelemetry-cpp |
| Tests | doctest, nanobench |

Deps build `-O2` even in dev, never with `-Werror`. Adding a dep needs a decision record.

## Guards (`scripts/lint/`)
| Guard | Fails on |
|---|---|
| `layering.sh` | Include across forbidden module boundary |
| `raii.sh` | `new`/`delete`/`malloc`/detached thread |
| `no_throw.sh` | `throw` in our code |
| `determinism.sh` | Clock/rand/unordered iteration in deterministic modules |
| `proto_compat.sh` | Removed/renumbered proto field (`buf breaking`) |
| `vectors.sh` | Object encoding change without new vectors |
| `ci_parity.sh` | `bin/check` and CI workflow drift |

## CI (GitHub Actions)
| Job | Runs |
|---|---|
| gate | `bin/check` on clang (`ci` preset) |
| gcc | build + unit on GCC 14 |
| sanitizers | `asan` preset: unit + integration + DST fast set |
| fuzz | each harness 60 s |
| web | `apps/web`: `x verify` |
| bench (main only) | compare to baseline, comment delta |
| images (main, tags) | multi-arch Docker → GHCR |

Rules: change detection (C++ change never triggers web, and vice versa); `concurrency: cancel-in-progress`; every job has a timeout; never merge red.
