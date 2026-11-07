# 🛬 Landing without a merge queue

> Land immediately with a compare-and-swap on the ref. Verify after the merge. Bisect and revert failures automatically.

## Why not a merge queue
A serial queue tests each change against the projected tip. Throughput is bound by CI capacity × flake rate. Agents retry flaky failures and amplify them. Agents also fix conflicts faster than humans find them, so cheap conflicts beat slow gates.

## Land algorithm
1. Client sends a **stack** of changesets, a **target ref** and an **idempotency key**. A retry returns the first result.
2. Server compares paths changed since the stack's base with paths in the stack.
   - **Disjoint** → rebase by replaying deltas. Cheap: only deltas move.
   - **Overlapping** → three-way merge on those files only. On failure, return a **conflict package** `(base, ours, theirs, paths)`.
3. CAS the ref with its version. Lost the race → rebase onto the new tip on the server and retry.

## Batching
- One **leader per ref** collects concurrent lands for a few ms, linearizes them and writes **one ref update per round**.
- Contention costs one transaction per round, not one per land. Matters on a Raft-backed store (YugabyteDB).
- Same idea as server-sequenced lockstep in games: the server stamps each command with a sequence number and applies a batch per tick.

## Conflict resolution plugin
- A resolver (agent, deterministic tool, or none) returns a proposal.
- A proposal lands only if a configured verification command passes on **the exact tree that would land**.
- Otherwise the author gets the package.

## Post-merge verification
- Verifier tests tips continuously.
- On failure: bisect through the history API (no clone), land an automatic revert of the culprit, notify the author.
- Flaky tests are quarantined. A test that fails then passes on the same tree is flaky by definition.

## Review gates
Approval rules (e.g. comment-only changes need fewer approvals) live in the review service. Cairn feeds it through the changelog watch API and enforces a "required approvals" ref policy at land time.

## States of a land
```
received → rebasing → (merging) → cas → landed
                         ↘ conflict (terminal, package returned)
              cas lost → rebasing (bounded retries) → failed: contention
landed → verified | reverted
```
Anything not listed is illegal.

Engine detail: [implementation/06-landing-engine.md](../implementation/06-landing-engine.md).
