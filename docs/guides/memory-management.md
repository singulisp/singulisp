# Memory Management Guide

Singulisp manages value lifetimes through ownership, borrowing, and allocation provenance rather than
a garbage collector. Choose between default global storage and explicit regions or arenas as appropriate
for the use case.

## Start with ownership

- If a single owner holds a value, use a Move type such as `String`, `Box`, or `Vec` directly.
- Borrow with `[& T]` for temporary reads and `[& mut T]` for exclusive updates.
- There is no `Rc` or `Arc` for unrestricted sharing of the same data. If stable shared identity is
  required, consider a generational handle from `std::pool`.
- Use `PVec` or `PMap` when immutable history is required.

## Default global provenance

```lisp
(let text "hello")
(let boxed (Box/new 42))
(let mut values (Vec/new))
```

Without an explicit arena, these values use global provenance. This is convenient, but it does not
eliminate the cost of frequent allocation and deallocation.

## Scratch arena

A bump arena is appropriate for discarding temporary values with the same lifetime as a group.

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

References and handles originating in scratch storage must not be returned outside the block. If the
required capacity is unknown, measure it before oversizing a fixed allocation, and consider `:linear`
or `:doubling` growth with an upper bound.

## Fixed-slot pool

Use a pool when repeatedly allocating values of the same size.

```lisp
(defregion @nodes
  :allocator (arena :pool 64)
  :lifetime :scoped
  :thread :local
  :default-capacity (MiB 2)
  :grow :fixed
  :on-oom :option
  :reuse :reset)
```

Values larger than the slot size and variable-length buffers cannot be stored. When selecting
`:on-oom :option` or `:result`, use fallible allocation APIs.

## Long-lived arena heap

Use an arena heap when individual Drop, persistent collections, FFI boundaries, or asynchronous holds are required.

```lisp
(defregion @state
  :allocator (arena :heap)
  :lifetime :drop
  :thread :local
  :allow-heap true
  :allow-persistent true
  :on-oom :result
  :reuse :release)
```

A region that uses `persist` requires both `(arena :heap)` and `:allow-persistent true`.

## Thread boundaries

Values originating in an arena with `:thread :local` cannot cross a thread boundary. A task may use a
`:lifetime :scoped` region when the heap arena is `:thread :async` and structured concurrency proves
that the join precedes region exit. Only a value held beyond the scope with `:async-hold true` requires
all of `(arena :heap)`, `:lifetime :drop`, and `:thread :async`. Borrowing conditions must also be met in every case.

## FFI boundaries

References from a scoped region cannot escape through FFI. Combine `:allow-ffi-out true` with
`:lifetime :drop`. The contract must state whether C retains the value and who releases it.

## Common mistakes

- Storing an arena-derived reference in a long-lived struct or collection
- Capturing a `:thread :local` value in a spawned task
- Placing variable-length storage such as `String` or `Vec` in a pool
- Specifying `:grow :fixed` and `:chunk-size` together
- Calling an infallible API with a fallible out-of-memory policy, or vice versa
- Using nonexistent attribute names such as `allow-arc`, `:lifetime :fixed`, or `:on-oom :return-err`

See [Regions and Allocation](../language/regions.md) for the complete list of attributes and combination constraints.
