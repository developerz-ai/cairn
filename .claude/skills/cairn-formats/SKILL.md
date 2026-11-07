---
name: cairn-formats
description: Cheat sheet for Cairn's canonical object encodings and ids. Use when reading or writing src/object, test vectors, or anything that hashes objects.
---
# Formats cheat sheet

- `id = BLAKE3-256(tag ‖ canonical_bytes)`; tags `cairn/blob/1`, `cairn/tree/1`, `cairn/cs/1`, `cairn/chunks/1`.
- Ids over **uncompressed** canonical bytes. zstd is storage only.
- Varint = unsigned LEB128, minimal. Bytes = varint len + bytes.
- Fixed field order. Collections sorted by raw bytes, no duplicates.
- Blob ≥ 1 MiB → chunk manifest (FastCDC 16/64/256 KiB).
- Tree entry = `(name, mode, child_id)`. Modes: file, exec, symlink, tree, submodule.
- Changeset = parents, change_id(16), author, committer, ts, tz, message, metadata map, delta sorted by path. **No root tree inside.**
- Decoder rejects trailing bytes, non-minimal varints, unsorted, unknown version, over-limit.
- Changing any byte = new tag version + decision record + vectors.

Full spec: `docs/implementation/02-object-formats.md`. Vectors: `tests/vectors/`.
