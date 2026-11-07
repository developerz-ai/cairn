# 🛤️ Roadmap

> Six milestones. Each gate must pass before the next starts. Each milestone becomes a GitHub milestone; work is planned in `docs/plans/`.

| # | Milestone | Scope | Gate |
|---|---|---|---|
| M0 | Formats | Canonical encodings, vectors, in-memory stores, `core` + `object` modules, fuzz harnesses | Round-trip + fuzz (24 h) clean; Python decoder matches vectors byte for byte |
| M1 | Single node | `cairn-server` on PostgreSQL + fs blobs; `CreateChangeset`, CAS land (no batching), sparse checkout, basic CLI; DST harness | Differential tests vs git agree; first benchmark published; wire frozen |
| M2 | History | Derived data framework, path history, blame, segments; git import | 100k-commit repo imports and answers `blame` without a local clone |
| M3 | Scale-out | YugabyteDB, S3, `cairnd`, `cairn-cache`, batch leader, ref leases | Multi-node DST + soak with 50 contending agents pass; ≥ 1,500 lands/min |
| M4 | Governance + forge v1 | Path ACLs, principals, redaction, `WatchChangelog`, verifier with bisect + auto-revert; `apps/web` browse, review, land feed | Security review; forge self-hosts Cairn's own repo |
| M5 | Virtual WC + git | FUSE overlay, git protocol v2 read path, read-only git mirror | Stock `git clone --filter` works against Cairn; FUSE build of a large repo |

## Dogfooding
From M4, Cairn's own repo is hosted on Cairn (mirrored read-only to GitHub).

## Risks
| Risk | Mitigation |
|---|---|
| Memory safety in parsers | Fuzzing, sanitizers, hardened libc++, small trusted core |
| Hot `main` ref on Raft store | Batching; measure in M3 before promising numbers |
| Scope creep into a full GitHub clone | Core stays APIs; forge phase 2 features need a decision record |
| Wire-format churn | Version every RPC and derived type; freeze objects at M0, wire at M1 |
| Name / license | Check before launch; credit prior art from day one |
