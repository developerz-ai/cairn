# 🧪 Testing

> Deterministic simulation is the backbone. Every bug becomes a seed or a fuzz reproducer committed to the repo.

## Tiers
| Tier | Scope | Budget | Runs |
|---|---|---|---|
| unit | Pure functions, encoders, merge, state machines | < 10 s total, all cores | every save, `bin/check` |
| integration | Real Postgres + Garage (containers), gRPC loopback | < 2 min | `bin/check`, CI |
| dst | Whole cluster in one process, simulated time/net/disk, N seeds | 200 seeds < 2 min | `bin/check` (fast set), nightly (10k seeds) |
| fuzz | Every decoder + RPC payload + path normalizer | 60 s each in CI | CI, nightly (hours) |
| differential | vs git: diff, merge, blame on real repos | < 5 min | CI |
| golden | Object vectors, CLI `--json` schemas | < 5 s | `bin/check` |
| bench | Micro + macro vs baseline | — | `bin/bench`, nightly |
| soak | 50+ agents landing for hours, multi-node | hours | nightly |

## Deterministic simulation testing (DST)
- One seed drives everything: `Rng::stream(seed, "net")`, `"disk"`, `"clock"`, `"workload"`. Adding a fault type doesn't reshuffle others.
- Simulated: clock (no `sleep`, time advances by event), network (latency, loss, reorder, partition), disk (latency, torn write, fsync lies), MetaStore (txn abort, serialization failure), process crash/restart.
- Components are state machines (`on(Event) → Effects`); the sim executes effects with a single-threaded event loop.
- **Invariant checks** after every step (ref versions monotonic, every landed change reachable, objects hash to ids, no lost acknowledged land). Violation → dump `seed + event log` and abort.
- Replay: `cairn-sim --seed 1234 --replay` reproduces exactly. `cairn-sim bisect` finds the first divergent step.
- Per-component state hashes (refs, queue, cache index) compared across replays.
- Workers = 0 (inline) mode for full determinism.

## Fuzzing
- libFuzzer harnesses in `tests/fuzz/<target>.cpp`; AFL++ compatible.
- Corpus in `tests/fuzz/corpus/<target>/`; crash fixes add the reproducer to `tests/fuzz/regressions/<target>/` (run in unit tier).
- Structure-aware mutators for trees and changesets.

## Property tests
`CAIRN_PROPERTY(name, gen)`: seed from test name; failure prints `seed=N`; `CAIRN_PROP_SEED` replays one; `CAIRN_PROP_RUNS` scales. Properties: encode∘decode = id; derive incremental = derive fresh; rebase of disjoint changes commutes; merge(a, b) = merge(b, a) when clean.

## Rules
- Fakes over mocks. Inject `Clock`, `Rng`, stores, transport.
- No `sleep` in tests; no wall-clock assertions in unit/integration.
- Every test has a timeout. Flaky = bug: quarantine with an issue, fix within a week.
- Unfixed repro: `test_suite("hunt")` + `may_fail`; stays visible, doesn't block.

## Framework
doctest (fast compile), one binary per module, ctest sharding. nanobench for micro benchmarks.
