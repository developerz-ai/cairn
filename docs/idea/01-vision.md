# 🎯 Vision

> A self-hostable GitHub alternative built for agents: source control where the server owns history, so thousands of agents can land changes on one monorepo without a merge queue.

As of 2026-11. Design stage.

## Problem
- Git puts the full history in every clone. A monorepo with hundreds of agents means hundreds of multi-GB clones, fetch negotiation CPU on the server, and minutes per checkout.
- Merge queues test each change against the projected tip. Throughput is capped by CI capacity and flake rate, and agents amplify both.
- History queries (log, blame, merge-base) need a local clone, so every CI job, bisect tool and agent pays for one.

## Users
| Who | Needs |
|---|---|
| Coding agents | Fast sparse checkout, land in seconds, conflict packages they can fix, `--json` everywhere |
| CI, verifier, bisect bots | History and diff over RPC, no clone |
| Humans | Web forge (GitHub-style): browse, review stacks, approve, watch lands, revert |
| Operators | One `docker compose up` to self-host; scale out by adding nodes |

## Core flows (happy path)
1. **Checkout**: client asks for a root tree, fetches missing trees and blobs within a sparse scope, writes files.
2. **Commit + land**: client uploads blobs, sends a stack and a target ref. Server rebases, merges disjoint paths, CAS-updates the ref.
3. **Conflict**: overlapping paths fail the three-way merge. Server returns a conflict package. An agent or resolver plugin fixes it and lands again.
4. **Verify**: verifier tests tips after merge, bisects failures through the history API, lands a revert.
5. **Query**: `log`, `blame`, `diff`, `grep` answered by the server from derived indexes.

## Principles
1. **Always online.** Protocol assumes a reachable server.
2. **Server computes, client lays out.** Server does history, blame, diff, merge, rebase. Client caches objects and writes files.
3. **Everything is an API.** Every CLI query is an RPC.
4. **Immutable and content-addressed.** Any cache answers any request.
5. **Cost scales with the change.** Commit touches the path spine, never the whole tree.
6. **Few refs.** `main` and release refs only.
7. **Checkouts are optional and sparse.**
8. **Self-hostable first.** One node on PostgreSQL + local disk works; no cloud service required.

## Success metrics
| Metric | Target |
|---|---|
| Warm sparse checkout, ~1,500 files | ≤ 0.2 s (CarterT27/scion: 0.45 s warm; git sparse: 0.21 s) |
| Lands/min, 50 contending agents, one 16-core node | ≥ 1,500 (gitai: 1,487–1,944) |
| Cache hit rate per tier | > 99% |
| Commit cost | O(changed files × tree depth) |

Targets to confirm with benchmarks after M1. See [prior art](02-prior-art.md) for the sources.

## Out of scope (v1)
- Offline work.
- Replacing git for general use.
- Two-way git sync (forces git's data model on writes).
- CI runners. Cairn triggers external CI and the verifier through webhooks and `WatchChangelog`.
- Issues, wikis, project boards in v1 (forge phase 2).

## GitHub alternative, split in two
| Layer | Owns | Language |
|---|---|---|
| **Cairn core** (`cairn-server`, caches, CLI) | Objects, history, landing, ACLs | C++23 |
| **Cairn forge** (`cairn-web`) | Orgs, repos list, review of stacks, approvals, land feed, verifier, admin | TypeScript on Ultimate |

Core never depends on the forge. Any other review tool can use the same APIs.

## Name
"Cairn": a stack of stones marking a trail. History as a pile of immutable stones. Working title; trademark check pending ([open questions](10-open-questions.md)).
