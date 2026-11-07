# 🔌 Protocol

> gRPC + protobuf between tiers and for tools. A thin HTTP/JSON gateway for the web forge. Plain `GET /o/<id>` for CDNs.

## Surfaces
| Surface | Who | Notes |
|---|---|---|
| gRPC (`proto/cairn/v1`) | CLI, `cairnd`, `cairn-cache`, CI, bots | Callback API server side; streaming for objects and `WatchChangelog` |
| HTTP/JSON gateway (`/api/v1`) | `cairn-web`, scripts | Same handlers, JSON codec. Bun has no first-class gRPC; gateway avoids it ([ADR 005](../decisions/005-grpc-plus-json-gateway.md)) |
| `GET /o/<id>` | CDN, `curl` | Immutable bytes, `Cache-Control: immutable, max-age=31536000`, ETag = id |
| Unix socket | CLI → `cairnd` | Same gRPC services over UDS |

## Services
```proto
service Objects { rpc GetTrees(stream TreeReq) returns (stream ObjFrame);
                  rpc GetBlobs(stream BlobReq) returns (stream ObjFrame);
                  rpc HasObjects(HasReq) returns (HasResp);
                  rpc PutBlobs(stream ObjFrame) returns (PutResp); }
service Write   { rpc CreateChangeset; rpc Rebase; rpc Merge; rpc Land; }
service History { rpc Log; rpc Blame; rpc MergeBase; rpc IsAncestor; rpc Diff; }
service Refs    { rpc ResolveRef; rpc WatchChangelog(WatchReq) returns (stream ChangelogEvent); }
service Locks   { rpc Lock; rpc Unlock; rpc ListLocks; rpc WatchLocks(WatchLocksReq) returns (stream LockEvent); }
service Search  { rpc FindFiles; rpc Grep; }
service Admin   { rpc SetAcl; rpc IssueToken; rpc Redact; rpc SetRefPolicy; }
```

## Rules
- **No negotiation.** Client asks for ids; server never computes "what you have".
- **Object frames** = `(id, kind, compressed zstd frame)`. Tiers pass frames through untouched (zero copy; `grpc::ByteBuffer` slices over cached memory).
- **Batching**: up to 1,000 ids or 16 MiB per message; flow control via HTTP/2 windows.
- **Idempotency**: every write takes `idempotency_key`; retries return the first result.
- **Errors**: one registry, stable codes (`CAIRN_CONFLICT`, `CAIRN_CAS_LOST`, `CAIRN_NOT_FOUND`, `CAIRN_FORBIDDEN`…). Forbidden paths return `NOT_FOUND`.
- **Deadlines** required on every call; server rejects calls without one from non-CLI principals.
- **Versioning**: package `cairn.v1`; fields append-only; removal = new package. Changelog at top of each `.proto`.
- **Auth**: bearer token (service principal / OIDC) in metadata; mTLS optional between tiers.

## Wire freeze
At M1 gate. Read CarterT27/scion `SPEC.md` first; reuse shapes where they fit.
