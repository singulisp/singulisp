# Traits

A trait defines an interface that a type must satisfy. It abstracts common behavior and enables generic programming and polymorphism.

---

## `deftrait` Syntax

### Basic Syntax

```lisp
(deftrait TraitName
  (method-name [parameter...] -> return-type)
  ...)
```

### Basic Example

```lisp
#[pub]
(deftrait Numeric
  (add [(self : Self) (other : Self)] -> Self)
  (sub [(self : Self) (other : Self)] -> Self)
  (neg [(self : Self)] -> Self)
  (zero [] -> Self))
```

- A method with `self` is an instance method.
- A method without `self`, such as `zero`, is an associated function.
- `Self` is replaced by the concrete type in an `impl`.

### Traits Without Parameters

A trait without type parameters may also be defined.

```lisp
(deftrait Printable
  (to-string [(self : Self)] -> String))
```

---

## `impl` Syntax

Write a concrete implementation of a trait with `impl`.

### Basic Syntax

```lisp
(impl TraitName TargetType
  (defn method-name [parameter...]
    body)
  ...)
```

### Implementation Example

```lisp
(defstruct Numeric3 :repr simd
  (x : f32) (y : f32) (z : f32))

(impl Numeric Numeric3
  (defn add [(self : Numeric3) (other : Numeric3)] -> Numeric3
    (Numeric3 (+ self.x other.x)
          (+ self.y other.y)
          (+ self.z other.z)))

  (defn sub [(self : Numeric3) (other : Numeric3)] -> Numeric3
    (Numeric3 (- self.x other.x)
          (- self.y other.y)
          (- self.z other.z)))

  (defn neg [(self : Numeric3)] -> Numeric3
    (Numeric3 (- 0.0f self.x)
          (- 0.0f self.y)
          (- 0.0f self.z)))

  (defn zero [] -> Numeric3
    (Numeric3 0.0f 0.0f 0.0f)))
```

### The Three Forms of `impl`

`impl` may be written in the following three forms. Each has the effect described below.

```lisp
; Form 1: explicit for keyword (recommended)
(impl Numeric for Numeric3
  (defn add [(self : Numeric3) (other : Numeric3)] -> Numeric3 ...))

; Form 2: shorthand with for omitted
(impl Numeric Numeric3
  (defn add [(self : Numeric3) (other : Numeric3)] -> Numeric3 ...))

; Form 3: method extension without a trait (inherent impl)
(impl Numeric3
  (defn length [(self : Numeric3)] -> f32 ...))
```

### Inherent Methods (`impl` Without a Trait)

Methods may also be added to a type without specifying a trait.

```lisp
(impl Numeric3
  (defn new [(x : f32) (y : f32) (z : f32)] -> Numeric3
    (Numeric3 x y z))

  (defn length [(self : Numeric3)] -> f32
    (f32/sqrt (+ (* self.x self.x)
                 (+ (* self.y self.y)
                    (* self.z self.z)))))

  (defn dot [(self : Numeric3) (other : Numeric3)] -> f32
    (+ (* self.x other.x)
       (+ (* self.y other.y)
          (* self.z other.z)))))
```

---

## Method Calls

Call trait methods and inherent methods with ordinary function-call syntax. Singulisp has no method-chaining (dot-call) syntax.

```lisp
(let v1 (Numeric3 1.0f 2.0f 3.0f))
(let v2 (Numeric3 4.0f 5.0f 6.0f))

; Trait method calls (ConcreteType/method form)
(let v3 (Numeric3/add v1 v2))
(let v4 (Numeric3/neg v1))

; Inherent method calls
(let len (Numeric3/length v1))
(let d (Numeric3/dot v1 v2))

; Associated function calls (without self)
(let origin (Numeric3/new 0.0f 0.0f 0.0f))
(let z (Numeric3/zero))
```

---

## Conventional Traits

Singulisp has no “built-in traits” automatically provided by the language prelude. The standard `Hash`, `Eq`, `Ord`, and `Iterable` traits are defined by the layer-2 `std` package and are explicitly imported with an ordinary `(use std::... [...])` form.

The compiler gives special treatment by name to four well-known traits: `std::hash::Hash`, `std::cmp::Eq`, `std::cmp::Ord`, and `std::iter::Iterable`. `Hash`, `Eq`, and `Ord` integrate with `#[derive]` to generate corresponding functions automatically for structs and enums. `Iterable` is a namespace used to resolve `Type/iter-len` and `Type/iter-at` statically in the non-Vec fallback of `for` desugaring; it cannot be derived. The names `Clone`, `GpuCompatible`, and `Serialize` receive no special treatment.

### Eq

Defines equality comparison. By convention, it is declared in the following form.

```lisp
(deftrait Eq
  (eq [(self : [& Self]) (other : [& Self])] -> bool))
```

`#[derive Eq]` generates a `Type/eq` function for the target type. For a struct, it returns true when all fields are equal. For an enum, it returns true when the variants match and every payload value is equal.

### Ord

Defines an ordering protocol. `<`, `>`, `<=`, and `>=` are exclusive to numeric types; an `Ord` implementation does not overload these operators for arbitrary types. Call the generated `Type/cmp` explicitly.

```lisp
(deftrait Ord
  (cmp [(self : [& Self]) (other : [& Self])] -> i32))
```

`#[derive Ord]` generates a `Type/cmp` function for the target type. Structs are compared lexicographically in field declaration order. Enums first compare variant declaration order, then compare the payloads of identical variants in declaration order.

### Hash

Defines hash computation. It is used by key protocols such as `std::collections::HashMap`.

```lisp
(deftrait Hash [T]
  (hash-into [(self : [& Self]) (h : u64)] -> u64)
  (hash [(self : [& Self])] -> u64))
```

`#[derive Hash]` generates `hash-into` and `hash` for the target type. For a struct, it also generates `keq`, which the built-in collection runtime uses for key equality; `keq` is not generated for enums.

In the table below, `TypeName` is an explanatory placeholder for the target concrete type name, not a reflection API. Because `keq` is a collection-runtime helper, do not call it directly; user code should use `Eq/eq` for equality comparison.

| Generated function | Description |
|---|---|
| `TypeName/hash-into` | Takes a seed and folds fields in declaration order |
| `TypeName/hash` | Returns the `u64` computed from a seed of 0 |
| `TypeName/keq` | Collection-runtime helper generated only for structs |

The generated value is a deterministic `u64`, but it is not a stable hash specification for sharing in a storage format or external protocol.

### Iterable

This protocol is used when `for` traverses a concrete type that does not match the Vec or range special case. An implementation provides `Type/iter-len` and `Type/iter-at`. As with the `for` induction variable, the index is `i32`.

### Clone

Defines value duplication.

```lisp
(deftrait Clone
  (clone [(self : Self)] -> Self))
```

It must be implemented manually with `impl`.

### GpuCompatible / Serialize

The compiler gives no special treatment to traits named `GpuCompatible` (GPU-buffer compatibility) or `Serialize` (BBin serialization). GPU compatibility and BBin serialization are handled through separate compiler-internal paths; using these names in `deftrait`, `impl`, or `#[derive]` assigns no language-level meaning.

---

## Trait Bounds (Generic Constraints)

A trait bound on a generic function's type parameter requires that type to implement a particular trait.

### Basic Syntax

In a generic function, place a **type-parameter vector** `[T ...]` immediately after the function name, followed by the ordinary parameter vector. Specify trait bounds with `:where` after the parameters and return type.

```lisp
(defn function-name [T] [(x : T)] -> return-type
  :where [[TraitName T]]
  body)
```

For a type parameter with a `:where` bound, a trait method may be called in the form `TraitName/method`; it is resolved to the concrete type through the bound.

### Example

```lisp
(deftrait Eq
  (eq [(self : Self) (other : Self)] -> bool))

(defn both-equal [T] [(a : T) (b : T)] -> bool
  :where [[Eq T]]
  (Eq/eq a b))
```

### Multiple Trait Bounds

```lisp
(defn sorted-unique [T] [(items : [Vec T])] -> [Vec T]
  :where [[Eq T] [Ord T]]
  ...)
```

---

## `impl Trait` Syntax

The `for` keyword may be omitted: `impl Trait Type` is shorthand for `impl Trait for Type`.

```lisp
; The following two forms are equivalent
(impl Numeric i32 ...)
(impl Numeric for i32 ...)
```

---

## Dispatch

Singulisp supports **static dispatch only**. Every trait-method call is resolved to a concrete type at compile time and is eligible for inlining. Dynamic dispatch is intentionally not provided, in accordance with the language policy of maximum speed and static monomorphization.

---

## `#[derive]` Targets and Generated Functions

`#[derive]` may be attached to `defstruct` and `deftype` (enums). It supports three traits: `Eq`, `Hash`, and `Ord`.

| Trait | Generated functions | Description |
|---|---|---|
| `Eq` | `TypeName/eq` | Tests equality by matching every field (for enums, matching variants and equal payloads) |
| `Hash` | `TypeName/hash-into` and `TypeName/hash` (structs also generate the internal helper `TypeName/keq`) | Generates chained-seed hashing and the hash from a seed of 0 |
| `Ord` | `TypeName/cmp` | Lexicographic comparison in field order (returns i32: -1/0/1) |

```lisp
#[derive Eq Hash]
(defstruct GridPos
  (x : i32)
  (y : i32))

#[derive Eq Ord]
(deftype Priority
  Low Normal High Critical)
; Ord: variant declaration order is the primary sort key → Low < Normal < High < Critical
```

The `Copy` trait cannot be derived. The compiler determines it automatically when all fields are Copy types.

---

## Trait Design Guidelines

1. **Keep traits small**: Include only closely related methods in a single trait.
2. **Use associated functions**: Define constructors and factory methods as associated functions without `self`.
3. **Use derive**: `Eq`, `Hash`, and `Ord` can be derived automatically with `#[derive]`. `Clone` must be implemented manually with `impl`.
4. **Domain objects**: Separate traits by behavior, such as `Readable`, `Writable`, and `Transformable`.
