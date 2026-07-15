# Ownership and Borrowing

Singulisp does not use garbage collection. It statically checks moves, copies, shared borrows, and mutable borrows of values. The region system supplements this with allocation provenance and lifetimes.

## Copy and Move

Value types are broadly divided into `Copy` and `Move` types.

The principal `Copy` types are:

- integers, floating-point numbers, `bool`, and `()`;
- the six named SIMD types;
- the ten built-in mathematical types;
- `[& T]`, `[& mut T]`, `StrSlice`, and `[Expr T]`; and
- structs, enums, and arrays whose every component is `Copy`.

The principal `Move` types are:

- `String`, `[Box T]`, `[Vec T]`, and SoA collections;
- persistent collections and `GpuBuffer`;
- `Reader`, `Writer`, and concurrency, module, and region handles; and
- closures and aggregates containing `Move` fields.

```lisp
(let a : i32 42)
(let b a)       ;; copy
(let c a)       ;; a remains available

(let text "payload")
(let moved text) ;; move
;; text is unavailable from this point onward
```

Whether a struct or enum is `Copy` or `Move` is determined by the ownership classification of its components.

## Shared Borrows

`(& value)` creates a shared reference `[& T]`. While a shared reference is live, its owner cannot be moved and the same place cannot be mutably borrowed.

```lisp
(defn sum [(values : [& [Vec i32]])] -> i32
  (loop [(i : i32 0) (total : i32 0)]
    (if (< i (Vec/len values))
      (recur (+ i 1) (+ total (Vec/get values i)))
      total)))

(let values (make-values))
(let total (sum (& values)))
```

When a reference to a collection element is required, explicitly borrow the place rather than the value-returning `Vec/get` itself.

```lisp
(defn first [(values : [& [Vec i32]])] -> [& i32]
  (& (Vec/get values 0)))
```

## Mutable Borrows

`(& mut value)` creates an exclusive `[& mut T]`. No other shared or mutable borrow of the same place may be created while it is live. The owner must also be declared with `let mut`.

```lisp
(defn append-zero [(values : [& mut [Vec i32]])] -> ()
  (Vec/push values 0))

(let mut values (Vec/new))
(append-zero (& mut values))
```

A borrow's live range is shortened to its last use, but a reference cannot be returned or stored somewhere that outlives its owner.

## Lexical Scope and Escape

A reference to an owner created in an inner scope cannot escape that scope.

```lisp
;; Compile-time error: outlives the local owner
(defn dangling [] -> [& String]
  (let local "temporary")
  (& local))
```

`StrSlice` is also a borrowed view. Although it is `Copy`, this does not mean it may be stored without restriction in return values, long-lived fields, `Vec`, `Box`, and similar locations. Materialize it with `String/from` when an owned string is required.

## Fields and Collections

Field access is tracked as a place. Borrows of distinct fields can sometimes be separated, but updates to multi-level places are restricted.

```lisp
(let mut point (Point 1 2))
(set! point.x 3)
```

`Vec/get` returns an element value, while `SoaVec/get-field` operates on the place of a specified field. Operations that would violate collection integrity by moving or borrowing an element are rejected.

## Region Provenance

Arena-backed `String`, `Box`, and `Vec` types may have an allocator parameter.

```lisp
[String @scratch]
[Box Node @scratch]
[Vec f32 @scratch]
```

Region handles and region-backed references cannot escape their region block. There is no implicit promotion between different provenances. For details, see [Regions](regions.md).

## Thread Boundaries

Borrowed references, thread-local arenas, and I/O, GPU, and module handles cannot cross a CPU task boundary. Fresh closures and function references, however, can have their captures checked at the boundary. Handles and owned values from a qualifying `:thread :async` arena may cross when the structured join is proven to occur before region exit. For details, see [Structured Concurrency](concurrency.md).

## unsafe

`unsafe` is an explicit syntactic boundary around unsafe operations. It is transparent to its body during IR lowering and does not relax borrow or ownership checks. It cannot be used as a raw-pointer type or as a general mechanism for bypassing checks.

## Drop and Trap

At an ordinary lexical-scope exit, cleanup for owned values is inserted. A runtime trap, by contrast, aborts without unwinding the stack. Destructors are not guaranteed to run on a trap. Manage external resources with this distinction in mind.
