---
paths: ["apps/web/**"]
---
# Forge (Ultimate app) rules

- Follow Ultimate conventions: `x g <kind>`, `x verify` green before done.
- Read Cairn data via `ctx.cairn` (gateway service). Never connect to Cairn's database.
- ACL decisions belong to Cairn. `policy` only gates screens.
- Forge state (reviews, comments, approvals, land feed) = Ultimate entities in the `cairn_web` database.
- Every user-facing action also `mcp: { expose: true }`.
- Doc: `docs/implementation/13-web-ui.md`.
