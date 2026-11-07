---
name: cairn-dst
description: How to write deterministic state machines and simulation tests in Cairn. Use when writing land, store, cache, net or derive code, or a DST test.
---
# Deterministic simulation

## Write components as state machines
```cpp
struct Thing { std::vector<Effect> on(const Event&); };   // no I/O, no clock, no rand
```
Effects are executed by an adapter: real (prod) or simulated (sim).

## Inject everything
`Clock&`, `Rng&` (`Rng::stream(seed, "net")`), `MetaStore&`, `BlobStore&`, `Transport&`.

## Writing a DST test
1. Build a cluster: `sim::Cluster c(seed, {.servers = 3, .faults = sim::Faults::all()});`
2. Drive a workload: `c.run(workloads::land_storm{.agents = 50}, 10'000 /*steps*/);`
3. Invariants are checked every step automatically; add specific ones with `c.check(...)`.
4. Failing seed prints `seed=N`; replay with `bin/sim --seed N --replay`.

## Rules
Never sleep. Never read real time. Sort before iterating hash containers. Parallel results merge in input order.

Doc: `docs/implementation/11-testing.md`.
