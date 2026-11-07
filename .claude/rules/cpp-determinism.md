---
paths: ["src/core/**", "src/object/**", "src/derive/**", "src/merge/**", "src/land/**", "src/sim/**"]
---
# Determinism rules

These modules must replay bit-for-bit under DST.

- No `std::chrono::*_clock::now()` — use injected `Clock&`.
- No `rand`, `<random>` engines, `random_device` — use `Rng::stream(seed, "name")`.
- No iteration over `unordered_*` when order reaches output, encoding, hashing or replies. Sort first.
- No pointer-value ordering. No thread ids in logic.
- State machines: `on(Event) -> Effects`. No I/O inside.
- Parallel work merges results in input order.
- Exceptions to a rule: `// cairn-allow-<rule>: <reason>`. Guard: `scripts/lint/determinism.sh`.
- Doc: `docs/implementation/11-testing.md` (DST).
