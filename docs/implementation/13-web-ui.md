# 🖥️ Web forge on Ultimate

> `apps/web` is an Ultimate app. It talks to `cairn-server` over the HTTP/JSON gateway, keeps forge state (reviews, comments, org settings) in its own database, and never reads Cairn's tables.

## Scaffold
```bash
cd apps && bunx create-ultimate web && cd web && bun run setup && bun run dev
```

## Mapping to Ultimate primitives
| Cairn need | Primitive |
|---|---|
| `CairnService` (gateway client, `ctx.signal`, deadlines) | `defineService` |
| Browse, file, log, blame, diff pages | `route` (`render: 'stream'`) whose `load` calls read `action`s |
| `getTree`, `getBlob`, `getLog`, `getBlame`, `getDiff` | `action` (read), `mcp: { expose: true }` |
| Raw file download | `query` with `download` proxying `GET /o/<id>` |
| Review, comment, approval | `entity` + `mutator` (optimistic) |
| Land from UI, revert, retry verify, start bisect | `action` → Cairn RPC; `job` with `step.waitForEvent` for completion |
| Land feed, verifier status | `task` consuming `WatchChangelog` (via gateway stream) → `job` writes `land_event`, `verifier_run` entities → live `query` |
| Screen access | `policy` using roles mapped from Cairn principals |

## Identity
- Cairn is the identity authority for repo access. The forge signs in users (OIDC or local), then exchanges for a Cairn token scoped to the user.
- Ultimate `Actor` kinds map 1:1: `user`, `service`, `agent`.
- ACL decisions are made by Cairn (forbidden = not found). The forge's `policy` only hides screens.

## Data
- Separate database on the same Postgres server (`cairn_web`), separate role. Ultimate needs `wal_level=logical` for live queries.
- Forge tables: `org`, `team`, `review`, `review_comment`, `approval`, `land_event`, `verifier_run`, `webhook`.
- Approvals are sent to Cairn as review attestations; Cairn's ref policy checks them at land time.

## UI quality
- Diff and blame rendered server-side, streamed; virtualized long files.
- Keyboard navigation (`j/k`, `g g`, `/` search), deep links to `path@changeset#Lnn`.
- Follow gold-standards design taste: one `DESIGN.md` in `apps/web/`.
