# Language Quick Tour

Singulisp is a statically typed S-expression language. It unifies predictable native code, explicit
memory management, and SIMD, SPMD, and GPU execution in a single language model.

## Values and types

Integer types are `i8` / `i16` / `i32` / `i64` and `u8` / `u16` / `u32` / `u64`.
Floating-point types are `f32` and `f64`. The language also provides `bool`, Unit `()`, and `String`.

```lisp
(let count : i32 42)
(let ratio : f32 0.25f)
(let enabled : bool true)
(let label : String "fast")
```

An unsuffixed floating-point literal is `f64`; one with the `f` suffix is `f32`. When the use context
does not determine an integer width uniquely, specify it with a type annotation or conversion function.

The following types are also built into the language for high-performance computing:

- SIMD: `f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, `u32x4`
- Mathematical types: `Vec2`, `Vec3`, `Vec4`, `Quat`, `Float3`, `Mat3`, `Mat3r`,
  `Mat4`, `Mat4r`, `Mat3x4`

```lisp
(let lanes (f32x4 1.0f 2.0f 3.0f 4.0f))
(let doubled (simd/add lanes lanes))
(let direction (Vec3 1.0f 0.0f 0.0f))
```

Names such as `Vec3` are reserved built-in types; you cannot define a `defstruct` with the same name.
See [Types](../language/types.md), [SIMD](../performance/simd.md), and
[Mathematical Types](../language/math-types.md) for details.

## Bindings and functions

`let` creates an immutable binding, while `let mut` creates a mutable binding. Functions are defined
with `defn`, and the last expression in the body is the return value.

```lisp
(defn clamp-count [(value : i32) (limit : i32)] -> i32
  (let mut result value)
  (when (< result 0)
    (set! result 0))
  (when (> result limit)
    (set! result limit))
  result)
```

A generic function places its type parameters immediately after the function name.

```lisp
(defn identity [T] [(value : T)] -> T
  value)
```

## Branching and iteration

`if` is an expression that always has an else branch. `match` destructures algebraic data types.
Iteration is provided by `loop` / `recur`, `while`, and `for` over ranges, `Vec`, or implementations of `Iterable`.

```lisp
(defn factorial [(n : i32)] -> i32
  (loop [(acc : i32 1) (i : i32 n)]
    (if (<= i 1)
      acc
      (recur (* acc i) (- i 1)))))
```

## Structs and enums

`defstruct` defines a product type, while `deftype` defines an algebraic data type.

```lisp
(defstruct Sample
  (time : f32)
  (value : f32))

(deftype ParseState
  Pending
  (Ready i64)
  (Failed String))

(defn state-ready? [(state : ParseState)] -> bool
  (match state
    [(Ready _) true]
    [Pending false]
    [(Failed _) false]))
```

## Ownership and collections

Scalars, SIMD values, mathematical types, and references are `Copy`. `String`, `Box`, `Vec`, and
persistent collections are `Move`. Shared references are written `[& T]`; exclusive mutable references
are written `[& mut T]`.

```lisp
(let mut values (Vec/new))
(Vec/push (& mut values) 10)
(Vec/push (& mut values) 20)
(let first (Vec/get (& values) 0))
```

When using an arena, you can make provenance explicit with `defregion` and `@R`. A default global
allocator is also available. See [Ownership](../language/ownership.md) and
[Regions](../language/regions.md) for details.

## Option and Result

Potentially failing operations use `[Option T]` or `[Result T E]`. `?` extracts the `Ok` value from a
`Result` and returns an `Err` to the caller.

```lisp
(defn positive [(value : i64)] -> [Result i64 String]
  (if (> value 0)
    (Ok value)
    (Err "positive value required")))

(defn sum-positive [(a : i64) (b : i64)] -> [Result i64 String]
  (let x (? (positive a)))
  (let y (? (positive b)))
  (Ok (+ x y)))
```

## Modules

The unit of a module is a folder. Multiple `.gu` files in the same folder are combined as source shards
that form a single module.

```text
src/main.gu              root module
src/helpers.gu           root module
src/codec/read.gu        codec
src/codec/write.gu       codec
src/codec/json/value.gu  codec::json
```

```lisp
(use codec [decode])
(use codec::json :as json)
```

See [Project Structure](project-structure.md) and [Modules](../language/modules.md) for details.

## Where to go next

- [Language Specification](../language/index.md)
- [Performance Model](../performance/index.md)
- [Standard Library](../standard-library/index.md)
