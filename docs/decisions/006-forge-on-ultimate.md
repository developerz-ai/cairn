# 006 — The forge (GitHub-alternative UI) is an Ultimate app with its own database
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
We want a self-hostable GitHub alternative. UI, review and admin are CRUD + realtime work that doesn't need C++.

## Decision
`apps/web` on Ultimate. Separate `cairn_web` database on the same Postgres server. Talks to Cairn through the gateway only. Cairn owns ACLs and history.

## Alternatives considered
- **Server-rendered UI inside cairn-server** — why not: slow to build, mixes concerns into the trusted core.
- **Fork Gitea/Forgejo** — why not: built around git's data model and per-repo permissions.

## Consequences
- Easier: auth, realtime, MCP tools, jobs for free.
- Harder: two languages in one repo; CI change detection required.
- Revisit when: Ultimate can't meet forge latency targets.
