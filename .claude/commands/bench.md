---
description: Benchmark a change against main with real numbers.
argument-hint: [bench name]
---
Use the `perf-engineer` subagent: run `bin/bench $ARGUMENTS` on `main` and on this branch, 5 runs each. Report a before/after table (p50, p99, throughput, allocs/request, RSS) and whether the change beats noise.
