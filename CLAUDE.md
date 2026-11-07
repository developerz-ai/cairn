# Cairn — agent contract

Self-hostable, server-centric version control + GitHub-alternative forge for agent-scale monorepos. C++23 core, TypeScript forge on Ultimate. **Status: design stage — docs only, no code yet.** Read the doc for your area before writing anything.

## Response rules
- Lead with the outcome. Short. No closing summaries.
- Say what you ran and what it printed. Never claim a check passed without running it.
- Unsure about a design point → check `docs/` first, then ask. Don't invent APIs or formats.

## Coding rules (C++)
- C++23. Errors are `std::expected<T, cairn::Error>`, `[[nodiscard]]`. **Never `throw`.** `CAIRN_ASSERT` for invariants.
- One owner per resource. No raw `new`/`delete`/`malloc`. No detached threads.
- Hot paths: **0 heap allocations** (request arena), no virtual calls in inner loops, no locks across `co_await`.
- Deterministic modules (`core object derive merge land sim`): no wall clock, no `rand`, no `unordered_*` iteration affecting output. Inject `Clock&`, `Rng&`.
- Files ≤ 500 LOC, functions ≤ 60 lines, one job per file.
- Every change: tests in the same PR. Every perf claim: before/after numbers.
- Full rules: `docs/implementation/08-cpp-standards.md`, `09-cpp-performance.md`, `10-memory.md`.

## Coding rules (web, `apps/web`)
Follow Ultimate's `CLAUDE.md` and `x verify`. Never read Cairn's tables; call the gateway. ACLs live in Cairn.

## Layout (planned)
| Path | What | Stack |
|---|---|---|
| `src/core` | errors, ids, arena, hash, varint, rng, clock | C++ |
| `src/object` | canonical encodings (frozen at M0) | C++ |
| `src/store` | MetaStore/BlobStore + backends (mem, sqlite, pg, yb, fs, s3) | C++ |
| `src/derive` `src/merge` `src/land` | derived data, diff/merge, land engine | C++ |
| `src/net` `src/server` `src/cache` `src/daemon` | gRPC adapters, binaries | C++ |
| `src/wc` `src/cli` | working copy (sparse, VFS), CLI | C++ |
| `src/sim` | deterministic simulation harness | C++ |
| `proto/cairn/v1` | wire contract, append-only | protobuf |
| `apps/web` | forge | Ultimate (Bun, SolidJS) |
| `docs/` | idea/, implementation/, decisions/, plans/ | md |

Layering: `core ← object ← store ← derive ← merge ← land ← server`. Adapters (`net`, `store/*`) are the only place heavy third-party headers appear.

## Commands (planned; not yet present)
| Command | Does |
|---|---|
| `bin/setup` | toolchain + deps + compose up Postgres + Garage (S3) |
| `bin/build [preset]` | CMake + Ninja |
| `bin/test [filter]` | unit tests (< 10 s) |
| `bin/check` | full gate ≡ CI. Run once before push |
| `bin/fmt` | clang-format + biome |
| `bin/bench` | benchmarks vs baseline |
| `bin/sim --seed N` | DST run / replay |
| `bin/fuzz <harness>` | libFuzzer |

## The loop
1. Read the owning doc. 2. Write the test. 3. Smallest change that passes. 4. `bin/test <filter>`. 5. `bin/check` before push. Red → fix, never skip.

## Hard boundaries
- Object encodings and `proto/` are contracts. Changing them needs a decision record + new test vectors.
- New third-party dependency → decision record in `docs/decisions/`.
- No secrets in code, logs, tests or docs.
- Don't push to `main`; PRs only. Conventional commits.

## Preventive rules (mechanism → guard)
| Rule | Why | Guard |
|---|---|---|
| Verify object id on every read | Caches are untrusted | unit + DST invariant |
| Forbidden path = not found | No existence leaks | ACL tests |
| Untrusted sizes checked before allocating | Zstd/len bombs | fuzz harnesses |
| Land is idempotent by key | Retries must not double-land | DST |
| Derived data is a pure function | Rebuildable, cacheable | property test fresh == incremental |

## Docs index
- Idea (what/why): `docs/idea/README.md`
- Implementation (how): `docs/implementation/README.md`
- Decisions: `docs/decisions/README.md`
- Plans: `docs/plans/README.md`
- Agents, skills, commands: `.claude/`

## What NOT to build
Offline mode · two-way git sync · CI runners · merge queue · per-agent server branches · ORM · exceptions · our own crypto.
