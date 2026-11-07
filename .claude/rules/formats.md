---
paths: ["src/object/**", "tests/vectors/**", "proto/**"]
---
# Formats and wire contract

- Object encodings are **frozen** after M0. Any byte change = new type tag version (`cairn/tree/2`) + decision record + new vectors.
- Decoders reject: trailing bytes, non-minimal varints, unsorted/duplicate entries, unknown versions, over-limit lengths.
- Every decoder has a fuzz harness in `tests/fuzz/`.
- `proto/cairn/v1`: fields append-only; never renumber or reuse; removal = new package. Update the changelog comment at the top.
- Errors use the registry codes (`CAIRN_*`); forbidden returns `NOT_FOUND`.
- Docs: `docs/implementation/02-object-formats.md`, `04-protocol.md`.
