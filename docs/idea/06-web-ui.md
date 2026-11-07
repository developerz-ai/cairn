# 🖥️ Web UI: the forge

> The GitHub-alternative half of Cairn. A web app built on [Ultimate](https://github.com/developerz-ai/ultimate) (Bun + Postgres + SolidJS). Cairn stays the source of truth; the UI is a thin client over its API.

## Screens
| Screen | What it shows |
|---|---|
| Orgs and repos | Repo list, teams, members, settings |
| Review | Stacks by change id, per-changeset diff, inline comments, approvals, "land" button |
| Repo browser | Tree at any ref or changeset, sparse-scope aware, ACL-filtered |
| File view | Contents, syntax highlight, raw download |
| History | `log` for repo or path, change ids, agent/task metadata |
| Blame | Per line, follows copies and renames |
| Diff | Any two changesets, path filter, side by side |
| Land feed | Live stream of lands, conflicts, retries, batch rounds |
| Verifier | Tip status, bisect progress, auto-reverts, quarantined tests |
| Agents | Lands per agent, conflict rate, revert rate |
| Admin | Path ACLs, service principals, tokens, redaction, ref policies |
| Health | Cache hit rates per tier, land latency, store status |

## Principles
- **No second source of truth.** ACLs, refs and history live in Cairn. Ultimate `policy` only gates screens, using roles from Cairn's identity.
- **Read through the API.** The UI never reads Cairn's tables directly.
- **Live where it matters.** Land feed and verifier update in real time.
- **Every action is also an MCP tool**, so agents can use the UI's operations (revert, retry verify, start bisect).
- **Same look of quality as the CLI**: fast, keyboard-first, no spinner walls.

## GitHub parity map
| GitHub | Cairn forge | Phase |
|---|---|---|
| Repos, orgs, teams | Repos, orgs, teams; path ACLs instead of per-repo perms | 1 |
| Pull requests | Stacks reviewed by change id; land = CAS, no merge button queue | 1 |
| Required reviews, CODEOWNERS | Ref policies + path owners, semantic approval rules | 1 |
| Actions / checks | External CI + verifier via webhooks; status shown per changeset | 1 |
| Code search | Trigram `Grep`/`FindFiles` RPCs | 2 |
| Issues, projects, wiki | Later; Ultimate entities | 2 |
| Releases | Release refs + artifacts in object store | 2 |

Review data (comments, approvals) is forge state in the forge's own database. Approvals reach Cairn as a land-time ref policy check.

Detail: [implementation/13-web-ui.md](../implementation/13-web-ui.md).
