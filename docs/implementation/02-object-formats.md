# 🧬 Object formats

> Canonical, hand-written binary encoding. Frozen at M0. Test vectors in `tests/vectors/` so a second implementation can match byte for byte.

## Id
```
id = BLAKE3-256( tag ‖ canonical_bytes )
tag: "cairn/blob/1" | "cairn/tree/1" | "cairn/cs/1" | "cairn/chunks/1"
```
- 32 bytes binary; hex in text, `b3:` prefix optional. Abbreviations ≥ 12 hex chars, server-resolved.
- Ids are computed over **uncompressed** canonical bytes. Compression is a storage detail.

## Encoding rules
- Unsigned LEB128 varints; no signed values in hashed objects (timestamps = unsigned µs since epoch + separate tz offset varint with bias).
- Byte strings = varint length + bytes. UTF-8 validated for names and messages; paths are bytes with a validated subset (no `/` in names, no NUL, no `.`/`..`).
- Fields in fixed order, no optional-field tags; absent = explicit zero length.
- Collections sorted by raw bytes; duplicates rejected.
- Decoder rejects: trailing bytes, non-minimal varints, unsorted entries, unknown versions, lengths over limits.

## Blob
- `< 1 MiB`: `tag ‖ bytes`.
- `≥ 1 MiB`: chunk manifest `tag "chunks" ‖ total_size ‖ [(chunk_id, size)]`. Chunks via FastCDC (min 16 KiB, avg 64 KiB, max 256 KiB; tunable before freeze).

## Tree
```
entry_count
[ name, mode(varint: file|exec|symlink|tree|submodule), child_id(32) ] sorted by name
```
Phase 2 (prolly): directories over a threshold split at content-defined boundaries (rolling hash over entry keys) into a B-tree-like node hierarchy. History-independent: same entries → same tree id regardless of edit order.

## Changeset
```
parents[]            # first parent = delta base
change_id(16)        # stable identity across rebases
author, committer    # (name, email, principal_id)
timestamp, tz
message
metadata             # sorted map: agent_id, task_id, model, ...
delta[]              # sorted by path: (path, op: add|modify|delete|rename_from, mode, blob_id)
```
- Root tree is **not** in the changeset; it's derived (so derivation can be upgraded).
- Signature (Ed25519) is stored next to the changeset, not inside the hashed bytes.

## Compression at rest
- zstd level 3 default; per-kind trained dictionaries (trees, changesets, small blobs).
- Frame header carries dictionary id. Decoder caps output size (anti-bomb: max declared size, max ratio).
- Cache tiers store and ship the **compressed frame** as-is: zero recompression on the hot path.

## Freeze checklist (M0 gate)
- [ ] Vectors for every kind, edge cases (empty tree, max path, unicode names, 1 MiB boundary).
- [ ] Fuzzed decoders, 24 h clean.
- [ ] Second decoder (Python, stdlib + blake3) matches all vectors.
