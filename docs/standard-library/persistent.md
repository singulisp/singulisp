# Persistent Collections

`PVec` and `PMap` are built-in collections that return a new version while preserving
the version that existed before an update. `PVec` uses a 32-way vector trie, while
`PMap` uses a bitmap-compressed HAMT for structural sharing.

Elements in `PVec`, and keys and values in `PMap`, may use any ownable type, including
owned values, user-defined aggregates, and nested collections.

## PVec

```lisp
(let v0 : [PVec i64] (PVec/new))
(let v1 (PVec/push v0 (to-i64 10)))
(let v2 (PVec/push v1 (to-i64 20)))
```

| Operation | Meaning |
|---|---|
| `PVec/new` | Empty vector |
| `PVec/len` | Number of elements |
| `PVec/get` | Value at an index |
| `PVec/push` | New version with an element appended |
| `PVec/set` | New version with an index updated |

An out-of-range `PVec/get` returns the element type's zero representation. An
out-of-range `PVec/set` retains and returns the original version.

`v0`, `v1`, and `v2` remain independently usable and share every node not affected by
an update.

## PMap

```lisp
(let m0 : [PMap i64 i64] (PMap/new))
(let m1 (PMap/assoc m0 (to-i64 1) (to-i64 100)))
(let m2 (PMap/assoc m1 (to-i64 2) (to-i64 200)))
```

| Operation | Meaning |
|---|---|
| `PMap/new` | Empty map |
| `PMap/len` | Number of entries |
| `PMap/assoc` | New version with a key inserted or updated |
| `PMap/contains?` | Tests whether a key exists |
| `PMap/get` | Returns the value for a key directly |
| `PMap/nth-key` / `PMap/nth-value` | Access by an `i32` index in HAMT DFS order |

`PMap/get` does not return `Option`. If the key is absent, it returns the value type's
zero representation. Call `PMap/contains?` first when absence must be distinguished
from a zero value.

`nth-key` and `nth-value` are O(n) enumeration helpers. If the index is out of range,
they return the key or value type's zero representation. Do not rely on their order as
a stable external protocol.

## SoaPVec and SoaPMap

`[SoaPVec T]` and `[SoaPMap K V]` are SoA variants with a separate persistent buffer
for each field. Fields, keys, and values may use any ownable type, including owned
values and nested collections. `SoaPVec` provides `push`, `set-field`, and `scatter`
for updates, and `get-field`, `gather`, and `len` for reads. `SoaPMap` provides `new`,
`assoc`, `get-field`, `gather`, `contains?`, and `len`.

## persist / thaw / freeze

`persist`, `thaw`, and `freeze` perform type-directed, structural conversion between
mutable and persistent representations.

```lisp
(persist @region value)
(thaw @region allocator resolver value)
(freeze value)
```

- `persist` converts a `Vec` into a persistent collection and recursively projects
  structs and object graphs.
- `thaw` restores a persistent representation into a mutable representation in the
  specified region.
- `freeze` converts a mutable representation into an immutable representation.

## Choosing a Representation

- Use `Vec` or `HashMap` when destructive updates and locality matter.
- Use `PVec` or `PMap` when old versions and structural sharing matter.
- Use an SoA variant when contiguous access by field matters.
- Use [BBin serialization](../language/serialization.md) to store or transfer values as
  byte sequences.
