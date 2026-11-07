# 004 — Land by compare-and-swap with server-side batching, verify after merge
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
Merge queue throughput is bound by CI capacity × flake rate; agents amplify both. Anthropic reports a proper merge queue doesn't make sense at agent scale.

## Decision
Server rebases/merges, CAS-updates the ref; one batch leader per ref writes one update per round. Post-merge verifier bisects and reverts.

## Alternatives considered
- **Serial merge queue** — why not: CI-bound throughput.
- **Speculative/batched merge queue (Bors-style trains)** — why not: still CI-bound; complex failure handling.

## Consequences
- Easier: lands in milliseconds; throughput bound by DB, not CI.
- Harder: `main` can be briefly red; needs reliable bisect + revert and flake quarantine.
- Revisit when: revert rate > 2% of lands for a sustained week.
