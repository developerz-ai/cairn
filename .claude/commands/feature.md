---
description: Implement a planned slice end to end — code, tests, review, docs.
argument-hint: <path to slice .md>
---
Implement slice: $ARGUMENTS

1. Read the slice and its `overview.md`. If anything is ambiguous, ask once, then proceed.
2. `cpp-implementer` (or `web-forge` for `apps/web`) implements it with tests.
3. If the slice touches land/store/cache/net/derive → `dst-hunter` runs 1000 seeds.
4. If it touches decoders/auth/ACL/paths → `security-auditor`.
5. If it touches a hot path → `perf-engineer` posts before/after.
6. `reviewer` on the diff. Fix blockers/majors, re-review.
7. `docs-keeper` updates docs. Update the slice status in `status.md`.
8. `bin/check` green. Report: files, tests, numbers, findings fixed.
