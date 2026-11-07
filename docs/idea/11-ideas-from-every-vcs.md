# 🧲 Best ideas from every VCS

> Cairn borrows the best idea from each system and fits them into one model: one content-addressed store, one server-owned history, one land path.

As of 2026-11. Sources in [prior art](02-prior-art.md); unverified items marked †.

## Taken
| From | Idea | Where it lands in Cairn |
|---|---|---|
| **Git** | Content-addressed Merkle DAG; cheap branches as pointers; protocol v2 `filter` | Object model; git façade (M5) |
| **Mercurial / Sapling** | Revsets; segmented changelog; stacks as first-class; ISL-style stack UI | `log` query language; `segments`; forge review |
| **Mononoke (Bonsai)** | Changeset = delta vs parent; everything else derived | Data model |
| **Jujutsu** | Stable **change id** across rewrites; operation log (undo any op) | Changeset `change_id`; per-user op log in `cairnd` |
| **Google Piper** | Trunk-based monorepo; OWNERS files at every dir; release branches with few cherry-picks | Ref policy; path owners as land gate |
| **Google CitC** | Workspace = base commit + tiny delta (avg < 10 files), stored server-side | Virtual WC overlay synced to server (optional) |
| **Google TAP** | Post-submit testing; **flake-aware** culprit finding (Bayesian bisect) | Verifier keeps per-test flake prior; bisects batches |
| **Google Rosie** † | Split large-scale changes by OWNERS, land independently | `cairn lsc split` for agent-made sweeping changes |
| **Google Critique** † | Review state machine: attention set, LGTM + approval separate | Forge review model |
| **Perforce Helix Core** | Typemap (`binary+l`, `+S<n>` keep-last-N); exclusive checkout; obliterate/archive; proxies near studios | [File types](12-file-types.md); `cairn-cache` = proxy |
| **Plastic / Unity VCS** | **Smart locks**: lock held until change reaches the destination branch; semantic merge; Gluon partial workflow for artists | Lock released when change lands on `main`; merge drivers; file-level sparse |
| **Git LFS** | `lockable` attribute → read-only until locked | Typemap `lockable`; server enforces at land |
| **Anchorpoint** | Auto-lock on first edit; live lock notifications in the editor | `WatchLocks` stream; VFS auto-lock on write |
| **Diversion** | Workspace state synced live → early conflict warnings | "Someone is editing this" signal from overlays |
| **Epic Lore** | Centralized, content-addressed, chunked large files, sparse on-demand workspaces, SDKs in many languages | Same core shape; we add CAS landing, forge, agent scale, locks today |
| **Unreal UGS / Horde** | Sync code + **prebuilt binaries** for a known-good commit | Derived artifact link per changeset; `cairn sync --green` |
| **Unreal One File Per Actor** | Tens of thousands of tiny files | Small-object packing; reference-integrity land hook |
| **Dolt** | Prolly trees: history-independent, diff ∝ change | Large directories |
| **Pijul / Darcs** † | Patches commute; conflicts are first-class states | Conflict package is data; disjoint lands commute by construction |
| **Fossil** † | Repo includes tickets, wiki, forum — one self-hosted binary | Forge phase 2; single-box install |
| **Subversion** † | Path-based authz; partial checkout of any subdir; global revision numbers | Path ACLs; sparse scope; changelog `seq` per ref |
| **Bazel REAPI** | CAS + Directory trees addressable by digest | Optional REAPI CAS endpoint: build from a commit with no checkout |
| **VFS for Git → Scalar** | Virtualization is fragile on macOS/Windows; cone sparse + fsmonitor gets close | Sparse mode stays first-class, VFS optional |
| **Anthropic Scion** | Server-side commits; cache tiers > 99% hit; Spanner CAS; bisect/revert pipelines | The overall design |

## Left out on purpose
| Idea | Why not |
|---|---|
| Distributed peer-to-peer history (git, hg) | Server owns history; offline is a non-goal |
| Global repository lock / serial merge queue | CI-bound; doesn't scale to agents |
| Per-repo permissions only (GitHub, Gitea) | Monorepo needs path ACLs |
| File locking enforced only client-side (LFS) | Unenforced locks are suggestions |
| Two-way git sync | Forces git's model on writes |
