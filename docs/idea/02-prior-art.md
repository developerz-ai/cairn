# 📚 Prior art

> Keep history and writes on the server, fetch immutable objects lazily, and derive every expensive structure from a small core log.

Verified 2026-11-06 against primary sources unless marked. Credit these projects in every public doc.

## Systems and papers
| Source | Adopt | Caveat |
|---|---|---|
| [Anthropic: Redesigning source control](https://www.anthropic.com/research/redesigning-source-control) (Scion, 2026-10-13) | Every commit/rebase/merge on the server; cache tiers > 99% hit each; CI checkouts two orders of magnitude faster; CAS on a single Spanner row; post-merge testing + automated bisect/revert pipelines | No code released. It says a "proper merge queue doesn't make much sense at agent scale", not that none exists. Updated 2026-10-15 to credit Mononoke/Bonsai |
| [Mononoke / Bonsai](https://github.com/facebook/sapling) | Changeset = changes vs parent; blame, history, manifests are derived | "Not yet supported for external usage"; GPL-2.0 |
| [Sapling segmented changelog](https://engineering.fb.com/2022/11/15/open-source/sapling-source-control-scalable/) ([code](https://github.com/facebook/sapling/tree/main/eden/scm/lib/dag)) | Graph as linear segments; ancestry in O(merges); lazy history | Server side not usable outside Meta |
| [EdenFS](https://github.com/facebook/sapling/blob/main/eden/fs/docs/Overview.md) | On-demand files: FUSE (Linux), NFSv3 (macOS), ProjFS (Windows) | Not supported for external use |
| [Google Piper + CitC](https://research.google/pubs/why-google-stores-billions-of-lines-of-code-in-a-single-repository/) (CACM 2016) | Spanner-backed monorepo; cloud workspaces holding only changed files; trunk-based | Proprietary |
| [Jujutsu](https://docs.jj-vcs.dev/latest/technical/architecture/) | Pluggable backends; **change id** separate from commit id | — |
| [Dolt prolly trees](https://www.dolthub.com/docs/architecture/storage-engine/prolly-tree) | History-independent, content-chunked trees; diff ∝ change | Higher write cost than B-tree |
| [VFS for Git](https://github.com/microsoft/VFSForGit) → [Scalar](https://git-scm.com/docs/scalar) | Cache servers near clients; focus on files that matter | Microsoft recommends Scalar for new deployments |
| [Git protocol v2](https://git-scm.com/docs/gitprotocol-v2) | `filter` (partial clone), `packfile-uris` — base for a git façade | Blobs still fetched on first read |
| [Bazel Remote Execution API](https://github.com/bazelbuild/remote-apis) | Digest-addressed CAS, ByteStream for large blobs | SHA-256 by convention |
| [YugabyteDB](https://docs.yugabyte.com/preview/architecture/transactions/transactions-overview/) | Raft-replicated sharded store, distributed txns, hybrid logical clocks, Postgres API | Measure cross-shard txn latency |
| [Software Engineering at Google, ch. 16](https://abseil.io/resources/swe-book/html/ch16.html) | Piper > 80 TB, 60–70k commits/workday; OWNERS at every dir; One Version; few long-lived branches | — |
| [TAP flake-aware culprit finding](https://research.google/pubs/flake-aware-culprit-finding/) (ICST 2023) | Bayesian bisect using per-test flake rates | — |
| [Epic Lore](https://github.com/EpicGames/lore) (2026) | Centralized, content-addressed, chunked large files, sparse on-demand workspaces; MIT | **Closest competitor.** Pre-1.0; scalable locking on roadmap |
| [Perforce typemap](https://www.perforce.com/perforce/r14.2/manuals/cmdref/file.types.html) | `+l` exclusive lock, `+S<n>` keep-last-N, obliterate/archive | Re-type needed for existing files |
| [Unity VCS smart locks](https://docs.unity.com/en-us/unity-version-control/smart-locks) | Lock held until change reaches destination branch | Locks don't cross replicas |
| [FastCDC](https://www.usenix.org/conference/atc16/technical-sessions/presentation/xia) (ATC 2016) | Content-defined chunking for large blobs | No maintained C++ lib; write ~200 lines |
| [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) | Fast SIMD hash for ids | — |

Full idea catalog (Google, Meta, Perforce, Plastic, Lore, LFS…): [11-ideas-from-every-vcs.md](11-ideas-from-every-vcs.md).

## Open Scion-style projects (2026-11-06)
None in C++. All independent and unaffiliated with Anthropic.

| Project | Stack | Notes |
|---|---|---|
| [feder-cr/git-ai](https://github.com/feder-cr/git-ai) ("gitai") | Go, MIT, Gitaly fork | v0.4.2. Real git underneath, changeset log in SQLite/Postgres, CAS landing with batching, resolver plugin, verify/bisect/revert. Single-machine tested; reports 1,487–1,944 lands/min |
| [CarterT27/scion](https://github.com/CarterT27/scion) | Rust, Apache-2.0 | gRPC, CockroachDB + S3, BLAKE3, zstd. Sparse checkout 0.45 s with warm daemon (0.95 s cold; git sparse 0.21 s). Read its `SPEC.md` before freezing our wire |
| [bnsd55/openscion](https://github.com/bnsd55/openscion) | Rust, Apache-2.0 | Server-side rebase landing, cache tiers, git import, fsck/gc |
| [tinyhumansai/openscion](https://github.com/tinyhumansai/openscion) | Rust, GPL-3.0 | — |
| [jmsbooth/openscion](https://github.com/jmsbooth/openscion) | MIT | Spec only |

## Where Cairn differs
- C++23 for memory layout, zero-copy streaming and I/O control.
- YugabyteDB and PostgreSQL first-class; same schema.
- Segmented commit-graph index and prolly directories from the start.
- Virtual filesystem is a core milestone, not deferred.
- Built for **every file type**, including multi-GB game assets with server-enforced locks (Lore has them on its roadmap).
- Ships a self-hostable forge (GitHub alternative) on Ultimate.

## Name check
No VCS named Cairn found. Crowded nearby: `cairn-dev/cairn` (agent system), a desktop git client "Cairn", `oritera/Cairn` (AI search). Decide before launch ([open questions](10-open-questions.md)).
