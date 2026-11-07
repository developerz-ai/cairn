# 002 — Object ids are BLAKE3-256 over a hand-written canonical encoding
Status: accepted
Date: 2026-11-06
Supersedes: —

## Context
Ids must be stable forever, identical across implementations, and fast to compute. Protobuf encoding is not canonical.

## Decision
`id = BLAKE3-256(type tag ‖ canonical bytes)`. Hand-written varint encoding, sorted fields, strict decoder, checked-in test vectors. Frozen at M0.

## Alternatives considered
- **SHA-256** — why not: 3–5× slower without SHA-NI; no tree-hash parallelism for big files.
- **Protobuf deterministic serialization** — why not: not guaranteed canonical across versions/languages.

## Consequences
- Easier: any cache can verify any object; second implementations can test byte for byte.
- Harder: every format change needs a new tag version.
- Revisit when: never for v1 objects; new kinds get new tags.
