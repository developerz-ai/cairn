# 005 — gRPC for tiers and tools, HTTP/JSON gateway for the web forge
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
Tiers need streaming and zero-copy frames. The forge runs on Ultimate (Bun), which has no gRPC client and refuses HTTP/2+protobuf dependencies.

## Decision
gRPC (callback API) between tiers and for CLI/CI. Same handlers exposed as `/api/v1` JSON in `cairn-server`. Plain `GET /o/<id>` for CDNs.

## Alternatives considered
- **gRPC client in Bun** — why not: dependency outside Ultimate's seams.
- **Connect protocol** — why not: no mature C++ server implementation found (2026-11).

## Consequences
- Easier: forge is a normal Ultimate app; scripts can use curl.
- Harder: two codecs to keep in sync — generated from the same `.proto`.
- Revisit when: a maintained C++ Connect server exists.
