# 🚀 Deploy and operate

> Containers only. Docker Compose for one box, Helm for a fleet. Same images everywhere.

## Images (GHCR, amd64 + arm64)
| Image | Binary | Notes |
|---|---|---|
| `cairn-server` | `cairn-server` | Also runs `migrate`, `fsck`, `backfill` subcommands |
| `cairn-cache` | `cairn-cache` | Needs memory + local NVMe |
| `cairn-web` | Ultimate app | `ROLE=web\|worker\|scheduler\|sync\|migrate` |
| `cairnd` | Node daemon | Usually a system package, image for k8s DaemonSet |

Base: distroless/static; binaries statically linked against libc++ and mimalloc. Tags: `vX.Y.Z`, `sha-<commit>`, `latest`.

## Compose (single node)
```yaml
services:
  postgres:  { image: postgres:17, command: ["-c", "wal_level=logical"] }
  server:    { image: ghcr.io/developerz-ai/cairn-server, ports: ["7420:7420"],
               environment: { CAIRN_META: "postgres://...", CAIRN_BLOBS: "file:///data/blobs" } }
  web:       { image: ghcr.io/developerz-ai/cairn-web, ports: ["3000:3000"],
               environment: { DATABASE_URL: "postgres://.../cairn_web", CAIRN_API: "http://server:7420" } }
```

## Config
One TOML file + env overrides (`CAIRN_*`). Validated at boot; unknown keys fail startup with the key name.

## Runtime contract
- `/livez`: process alive, no dependency checks. `/readyz`: MetaStore + BlobStore reachable; 503 while draining.
- SIGTERM: stop accepting, finish in-flight lands (bounded), release ref leases, exit.
- Rolling upgrade, `maxUnavailable: 0`.

## Observability
| Signal | Tool |
|---|---|
| Traces | OpenTelemetry (OTLP), one span per RPC, land round, derivation |
| Metrics | Prometheus: RPC latency, cache hit per tier, lands/min, CAS lost, queue depth, allocs/request, arena high-water |
| Logs | Structured JSON (quill), `request_id`, `principal`, `ref` |
| Profiling | Continuous `perf`/eBPF sampling in staging; frame pointers in release |

Alerts only on SLO breach: land p99, checkout p99, error rate, cache hit < 95%.

## Backups and recovery
- MetaStore: nightly `pg_dump` / YSQL snapshot + WAL/PITR; monthly restore drill.
- BlobStore: append-only sync to second bucket (objects immutable).
- `cairn-server fsck`: verifies refs → changesets → trees → blobs reachability and hashes.
