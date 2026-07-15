# Regions and Allocation

A region is a unified contract governing allocation storage, lifetime, thread boundaries, OOM behavior, and FFI. In addition to ownership and borrowing, Singulisp tracks allocation provenance and rejects values that escape or operations that violate the contract at compile time.

Ordinary `String`, `Box/new`, and `Vec/new` allocations have the default global provenance. To use an arena, explicitly declare a `defregion` and a region instance.

## Definition

```lisp
(defregion @scratch
  :allocator (arena :bump)
  :lifetime :scoped
  :thread :local
  :default-capacity (MiB 8)
  :grow :fixed
  :on-oom :trap
  :reuse :reset)
```

A name beginning with `@`, `:allocator`, and `:lifetime` are required. Other Boolean capabilities default to `false`; `:thread` defaults to `:local`; and `:on-oom` defaults to `:trap`.

## Storage policy

| `:allocator` | Meaning |
|---|---|
| `global` | Default heap with process lifetime |
| `(arena :bump)` | Bump pointer; reclaimed in bulk on reset or drop |
| `(arena :heap)` | Heap belonging to the arena; individual deallocation is possible |
| `(arena :pool N)` | Pool of fixed-size `N`-byte slots |

The slot size of a pool must be at least one byte. A pool cannot store variable-length buffers.

## Lifetime and Thread Policy

| Attribute | Value | Meaning |
|---|---|---|
| `:lifetime` | `:scoped` | Reclaimed in bulk at the end of the lexical scope |
| `:lifetime` | `:drop` | Reclaimed according to the Drop of owned values |
| `:thread` | `:local` | Thread-affine; the default |
| `:thread` | `:async` | May cross a thread boundary under a proof of structured join |

`:fixed` is not a lifetime value; it is the value in `:grow :fixed`. The thread policy takes exactly one value: either `:thread :local` or `:thread :async`.

## Capability

| Attribute | Meaning |
|---|---|
| `:allow-heap` | Allows global heap allocation from within the region |
| `:allow-persistent` | Allows persistent collections and `persist` |
| `:cap-suspend` | Allows suspension by `await` and similar operations |
| `:cap-block` | Allows blocking waits |
| `:allow-ffi-out` | Allows region-backed references to be passed to FFI |
| `:allow-ffi-in` | Allows placement of values received from FFI |
| `:async-hold` | Allows values to be retained across `await` or a thread boundary |

All of these are Boolean. `allow-arc` is not a region attribute.

## Capacity and Growth

| Attribute | Value |
|---|---|
| `:default-capacity` | Byte count, `(KiB N)`, `(MiB N)`, or `(GiB N)` |
| `:grow` | `:fixed`, `:linear`, or `:doubling` |
| `:chunk-size` | Unit of linear growth |
| `:max-capacity` | Maximum capacity |

`:default-capacity` is required for bump allocators and pools. `:grow :fixed` and `:chunk-size` cannot be specified together. A pool may either omit `:grow` or specify only `:fixed`.

## OOM and Reuse

| Attribute | Value | Meaning |
|---|---|---|
| `:on-oom` | `:trap` | Runtime trap |
|  | `:result` | Returns allocation failure as `Result` |
|  | `:option` | Returns allocation failure as `Option` |
|  | `:abort-scope` | Exits the entire arena scope as a failure |
| `:reuse` | `:reset` | Rewinds the cursor and reuses storage |
|  | `:release` | Releases storage |

`:return-err` and `:fallback-heap` are rejected. The OOM policy must agree with whether the allocation API is fallible or infallible.

## Principal Static Constraints

- An arena with `:scoped` cannot specify `:allow-persistent true`.
- `:allow-persistent true` requires `(arena :heap)`.
- `:lifetime :scoped` cannot specify `:allow-ffi-out true`.
- `:allow-ffi-out true` requires `:lifetime :drop`.
- `:thread :local` cannot enable `:cap-suspend`, `:cap-block`, or `:async-hold`.
- A `global` allocator or bump arena cannot use `:thread :async`.
- `:async-hold true` requires all of `(arena :heap)`, `:lifetime :drop`, and `:thread :async`.
- A reference backed by a scoped region cannot be stored in a value that outlives that region.

## Region Instances

```lisp
(@scratch arena
  (let value (arena/alloc! arena 42))
  (use-value value)
  (arena/reset! arena))
```

The handle of a region block and references backed by that region cannot be returned outside the lexical scope. A diagnostic is also issued if live tasks or live borrows remain at reset or drop.

## Types with Arena Provenance

Types that own allocations have allocator parameters where required.

```lisp
[String @scratch]
[Box Node @scratch]
[Vec f32 @scratch]
```

Provenance is carried into code generation and ownership checking. No implicit copy or promotion occurs between different address spaces or lifetimes.

## Persistent Boundary

`persist` may be used only in a region with both `:allow-persistent true` and `(arena :heap)`. `thaw` and `freeze` make the boundary between persistent and mutable representations explicit. For structural type projection, see [Persistent Collections](../standard-library/persistent.md).

## Diagnostics

Definition parse errors are reported as `E-RGN-01xx`, cross-attribute constraint violations as `E-RGN-Cxx`, and value escapes or allocation-site violations as region, borrow, or IR diagnostics.
