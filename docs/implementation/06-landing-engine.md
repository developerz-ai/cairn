# 🛬 Landing engine

> One batch leader per ref turns many concurrent lands into one CAS per round. Built as a deterministic state machine so DST can drive it.

## Shape
```
Land RPC ──▶ idempotency check ──▶ per-ref MPSC queue ──▶ leader loop (one per ref, one owner node)
                                                           │ every round (≤ 5 ms or N lands):
                                                           │ 1. read tip (version v)
                                                           │ 2. for each land in arrival order:
                                                           │      rebase / merge onto running tip
                                                           │      ok → append; conflict → reject
                                                           │ 3. one txn: changesets + root trees + refs CAS(v → v+1) + ref_log
                                                           │ 4. CAS lost → re-read, replay round
                                                           ▼
                                                  complete futures, publish changelog
```

## Leader placement
- Ref → owner node via rendezvous hashing over live servers (membership in MetaStore, leases with fencing tokens = ref version).
- Non-owners forward `Land` to the owner. Owner loss → lease expires → new owner; in-flight lands retried by idempotency key.

## Rebase and merge
- **Disjoint paths** (bitset/sorted-merge of path sets since base): replay delta → new changeset (same change id, new parents) → root tree by spine rewrite.
- **Overlap**: three-way merge per file (histogram diff, line-based; binary = conflict). Clean → merged blob; dirty → conflict package.
- Path sets for "changed since base" come from the `segments` + `path_history` indexes; cached per tip.

## State machine
```cpp
struct LandSm {                    // pure: no I/O, no clock, no rng
  std::vector<Effect> on(const Event&);   // Event: Received, TipRead, MergeDone, TxnResult, Timeout
};
```
- Effects (`ReadTip`, `Merge`, `CommitTxn`, `Reply`) executed by an adapter. Production adapter = real stores; DST adapter = simulated stores with injected faults.
- Same idea as railroads' `state' = step(state, cmds)`: one ordered log in, effects out, replayable from a seed.

## Limits and backpressure
| Limit | Default |
|---|---|
| Round window | 5 ms or 64 lands |
| Max stack size | 100 changesets |
| CAS retries per land | 8, then `CAIRN_CONTENTION` |
| Queue depth per ref | 10,000; beyond → `RESOURCE_EXHAUSTED` with retry-after |

## Metrics
`land_latency_ms` (p50/p99), `lands_per_round`, `cas_lost_total`, `conflicts_total`, `queue_depth`. Benchmark: 50 contending agents, target ≥ 1,500 lands/min on one 16-core node.
