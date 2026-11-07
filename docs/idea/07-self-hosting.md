# 🏠 Self-hosting

> Runs on one box with PostgreSQL and a disk. Grows to a multi-region fleet without changing the data model.

## Sizes
| Size | Topology | Stores |
|---|---|---|
| **Laptop / dev** | `cairn-server` + `cairn-web` in Docker Compose | PostgreSQL, local disk |
| **Small team** | One VPS, Docker Compose | PostgreSQL, local disk or S3-compatible (Garage, SeaweedFS, R2, B2, AWS) |
| **Org** | N stateless servers behind a load balancer, `cairnd` on build machines | PostgreSQL HA or YugabyteDB, S3 |
| **Fleet** | Regional `cairn-cache` per site, many server nodes, Kubernetes (Helm) | YugabyteDB multi-zone, S3 |

## Promises
- **No external service required.** No SaaS identity, no cloud API. OIDC is optional; built-in tokens work.
- **One binary per role**, static where possible. Multi-arch images (amd64, arm64).
- **Same schema** on PostgreSQL and YugabyteDB YSQL. Move up by migrating data, not code.
- **Backups**: `pg_dump` (or YSQL snapshots) + object store copy. Objects are immutable, so blob backup is append-only sync.
- **Health**: `/livez`, `/readyz`, Prometheus metrics, OpenTelemetry traces.
- **Upgrade**: rolling. Derived data versions are built beside old ones; clients never notice.

## Day-one experience
```bash
docker compose up -d          # server, web, postgres
cairn init --server http://localhost:7420
cairn import git ./my-repo    # one-way import
cairn checkout main --sparse src/
```
Target: from `git clone` of Cairn to a browsable imported repo in under 5 minutes.

Detail: [implementation/14-deploy-and-ops.md](../implementation/14-deploy-and-ops.md).
