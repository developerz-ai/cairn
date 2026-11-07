# 🗃️ Every file type

> Source code, 4 GB game assets, datasets, binaries: one store, one history. A versioned **typemap** decides how each path is stored, locked, diffed, merged and previewed.

## Typemap
Lives in the repo at `/.cairn/typemap.toml`; versioned like code. Changes apply to **existing** files too (fixes Perforce's re-type problem).

```toml
[[type]]
glob = "**/*.{uasset,umap}"
kind = "binary"
lockable = true            # exclusive edit; enforced at land
keep_last = 0              # 0 = keep all; N = purge content older than N revisions
chunking = "fastcdc"       # dedupe edits inside big files
delta = "zstd-patch-from"  # cold-tier delta vs previous revision
preview = "thumbnail"      # derived preview generator
merge = "none"             # none | text | <driver name>
integrity = "unreal-refs"  # optional land hook

[[type]]
glob = "**/*.{png,psd,exr}"
kind = "binary"; lockable = true; preview = "thumbnail"

[[type]]
glob = "**/*.{unity,prefab}"
kind = "text"; merge = "unity-yaml"

[[type]]
glob = "**/*"              # default
kind = "auto"              # sniff: NUL in first 8 KiB → binary
```

## Storage by size
| Size | Stored as |
|---|---|
| < 4 KiB | Inline in the tree's pack / small-object pack |
| < 1 MiB | One object |
| ≥ 1 MiB | FastCDC chunk manifest; chunks dedupe across revisions and files |
| ≥ 1 GiB | Same; client fetches only chunk ranges it reads (VFS) |
| Purged (`keep_last`, obliterate) | Tombstone: id + size kept, bytes gone, reads return `PURGED` |
| Archived | Bytes moved to cold storage class; rehydrated on read |

## Locking
- **Lockable** files are read-only in the working copy until `cairn lock <path>` (or auto-lock on first write in VFS).
- A lock is a right to the **next revision on the target ref**. Released when the change **lands** on that ref (Plastic smart locks), not on upload.
- Server enforces at land: changing a lockable file without holding its lock → rejected.
- Locks are rows in MetaStore; `WatchLocks` streams changes so editors grey out locked assets live.
- Admin break-lock is audited. Locks by agents expire with their token.

## Diff and merge
| Kind | Diff | Merge |
|---|---|---|
| Text | Line diff (histogram) | Three-way; conflict package |
| Structured text (JSON, YAML, Unity) | Driver | Driver in a server-side sandbox, verified before land |
| Binary, lockable | Metadata + preview side by side | Never; lock required |
| Binary, not lockable | Metadata + preview | Conflict on overlap; last-writer requires explicit `--take theirs/ours` |

Merge drivers run sandboxed (no network, CPU/memory caps) and must be deterministic.

## Previews
Derived data type `preview`: thumbnails for images/3D/Unreal assets, page 1 for PDFs, waveform for audio. Generated lazily by worker plugins; shown in the forge and in `cairn show`.

## Normalization and hygiene
| Issue | Rule |
|---|---|
| Line endings | Stored as committed; optional per-type `eol = "lf"` normalization at commit |
| Exec bit, symlinks | Tree modes `exec`, `symlink`; symlinks never followed on the server; escaping symlinks rejected at land |
| Case-fold collisions (macOS/Windows) | Case-folded path index; colliding add rejected at land |
| Unicode normalization | NFC-equivalent duplicate names rejected at land |
| Max file size | Per-type cap (`max_size`); default 16 GiB |
| Generated junk | Typemap `deny` globs (e.g. `**/Intermediate/**`) rejected at land |

## Game studio flow
1. Artist mounts VFS (or Gluon-style file-level sparse).
2. Opens `Hero.uasset` → auto-lock → editor shows it locked to others.
3. Saves; overlay holds the change. Integrity hook checks references.
4. `cairn land` → lock released when it hits `main`.
5. Programmers `cairn sync --green` to get code + prebuilt editor binaries for the last verified changeset.
