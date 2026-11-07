# 🧮 Derived data

> Every expensive structure is a pure function of a changeset and the same type's values for its parents. Versioned, rebuildable, computed once.

## Framework
```cpp
template <class T>
concept Derivable = requires(const Changeset& cs, std::span<const T> parents, DeriveCtx& ctx) {
  { T::kName }    -> std::convertible_to<std::string_view>;
  { T::kVersion } -> std::convertible_to<int>;
  { T::kEager }   -> std::convertible_to<bool>;   // on write, or lazily on first read
  { T::derive(cs, parents, ctx) } -> std::same_as<Task<expected<T, Error>>>;
};
```
- Concept + static dispatch, not virtual: types are known at compile time; no vtable on the hot path.
- Key: `(kind, version, changeset id)`. New version built beside old; switch when backfilled.
- **Single-flight**: concurrent derivations of one key collapse to one (sharded in-flight map of shared futures).
- **Recursion without stack blowup**: walk ancestors iteratively until a stored value, then derive forward in topological order.
- **Idempotent**: same inputs → same bytes. Property test: derive twice, compare; derive from scratch vs incremental, compare.

## Types
| Type | Eager | Value |
|---|---|---|
| `root_tree` | ✅ | Tree id: parent root + delta, rewriting only the path spine |
| `segments` | ✅ | Segmented changelog: linear runs split at merges; ancestry in O(merges) |
| `skiplist` | ✅ | Skip pointers for fast `log` paging and merge-base |
| `path_history` | ❌ | Rows into `path_hist` |
| `blame` | ❌ | Per-file-version line → origin changeset graph, incremental from parent's |
| `copies` | ❌ | Rename/copy detection by blob id + similarity |
| `trigram` | ❌ | Sharded trigram posting lists (roaring bitmaps) |
| `git_commit` | ❌ | Git commit sha for read-only mirror |

## Root tree: path spine rewrite
```
for each dir touched by delta (deepest first):
  load dir tree (cache) → apply entries → encode → id → put
parent dir picks up new child id → ... → root
```
Cost: O(changed paths × depth). `root_tree` trees are written in the land txn so reads never wait; lazy types write after commit.

## Workers
- Lazy derivations run on a work-stealing pool; eager ones on the land path.
- Backfill job per new version: topological order, checkpointed, rate-limited.
