# ❓ Open questions

Answered questions become [decision records](../decisions/README.md). Non-blocking ones proceed as `ASSUMPTION:`.

| # | Question | Blocks | Current assumption |
|---|---|---|---|
| 1 | Is "Cairn" free as a name / trademark for a VCS? | Public launch | `ASSUMPTION:` keep as working title |
| 2 | License: Apache-2.0 or MIT? | First external contribution | `ASSUMPTION:` Apache-2.0 (patent grant) |
| 3 | Bazel Remote Execution CAS endpoint in v1 or later? | M3 | Later add-on |
| 4 | YugabyteDB via YSQL, or its key-value layer if cross-shard txn latency is too high? | M3 | YSQL; benchmark the hot ref row first |
| 5 | Chunk sizes for FastCDC and prolly-tree thresholds | M2 | 64 KiB avg chunks; shard dirs > 4,096 entries. Needs real monorepo data |
| 6 | Reuse CarterT27/scion's wire protocol or define our own? | M1 wire freeze | Own protocol; read their SPEC.md first |
| 7 | gRPC from Bun for the web UI, or HTTP/JSON gateway on the server? | Web UI | HTTP/JSON gateway ([ADR 005](../decisions/005-grpc-plus-json-gateway.md)) |
| 8 | Windows client (ProjFS) in scope? | M5 | No; Linux + macOS first |
| 9 | Seastar for `cairn-cache` or plain io_uring + thread-per-core? | M3 | Plain io_uring; revisit if p99 misses target |
