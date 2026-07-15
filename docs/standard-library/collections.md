# Collections

Singulisp provides statically typed collection types. All collections support generics, and their element types are determined at compile time.

---

## Vec (Dynamic Array)

A variable-length array that stores elements in contiguous memory. The default form uses global provenance; a form with an arena parameter is also available.

### Type

```lisp
[Vec T]
```

`T` is the element type. For example, `[Vec i32]` is a dynamic array of 32-bit integers.

### Construction

```lisp
;; Create an empty Vec
(let mut v (Vec/new))

;; Construct by appending elements
(let mut v (Vec/new))
(Vec/push (& mut v) 1)
(Vec/push (& mut v) 2)
(Vec/push (& mut v) 3)
```

### Operations

#### `Vec/push` -- Append (Mutating)

```lisp
(Vec/push (& mut v) value)
```

Takes a mutable reference and appends an element. **The first argument must be a mutable reference of the form `(& mut vec)`**; passing an owned value is an error. The return value is `()`.

`Vec/push!` is an alias of `Vec/push`, but it does not add the mutable reference automatically. Write `(Vec/push! (& mut v) x)`.

Behavior on memory-allocation failure (OOM) depends on the region policy. Use the following variants to select behavior explicitly.

| Variant | Behavior on OOM |
|---------|-----------------|
| `Vec/push` | Follows the region policy (default: abort) |
| `Vec/push-trap` | Always aborts |
| `Vec/push-option` | Returns `(Some ())` on success and `None` on failure |
| `Vec/push-result` | Returns `(Err AllocOom)` on failure |

```lisp
(let mut v (Vec/new))
(Vec/push (& mut v) 1)
(Vec/push (& mut v) 2)
(Vec/push (& mut v) 3)
(Vec/push (& mut v) 4)
;; v is [1 2 3 4]

```

#### `Vec/get` -- Element Access

```lisp
(Vec/get (& v) index)
```

Returns the element at the specified index. The index must have an integer type; an unannotated literal defaults to `i32` in normal contexts. **No bounds check is performed. An out-of-range index causes undefined behavior (UB).** Unlike `Array/get`, which traps on an out-of-range index, `Vec/get` performs no check.

```lisp
(let mut v (Vec/new))
(Vec/push (& mut v) 10)
(Vec/push (& mut v) 20)
(Vec/push (& mut v) 30)
(let x (Vec/get (& v) 1))  ;; x = 20
```

`Array/get` and `Vec/get` accept integer indices, including an explicit `i64`.

#### Updating and Capacity

```lisp
(Vec/set (& mut v) index value)
(Vec/reserve (& mut v) capacity)
```

`Vec/set` is also unchecked, and an out-of-range index causes undefined behavior. The argument to `Vec/reserve` is the minimum desired capacity, not an additional amount. The vector grows to at least that value only when the requested capacity exceeds its current capacity. Values outside `0..i32::MAX` trap. A `Vec` with arena provenance uses `Vec/new-in` and the corresponding allocation APIs, which follow the region policy.

#### `Vec/len` -- Length

```lisp
(Vec/len (& v))
```

Returns the number of elements in the Vec as an `i32`.

```lisp
(let mut v (Vec/new))
(Vec/push (& mut v) 1)
(Vec/push (& mut v) 2)
(Vec/push (& mut v) 3)
(println! "Length: {}" (Vec/len (& v)))  ;; Length: 3
```

#### `Vec/fill` -- Fill Every Element with the Same Value (Mutating)

```lisp
(Vec/fill (& mut v) value)
```

Overwrites every element in `0..len` with `value` while preserving the current length. The length does not change. The return value is `()`. Because `value` is duplicated into every element, the element type must be `Copy`.

```lisp
(let mut xs : [Vec f64] (Vec/new))
(Vec/push (& mut xs) 1.0)
(Vec/push (& mut xs) 2.0)
(Vec/push (& mut xs) 3.0)
(Vec/fill (& mut xs) 4)
;; xs is [4.0 4.0 4.0]
```

### Iteration

Singulisp's `for` expression can iterate over a `Vec` directly. An owned `Vec<T>` binds each element as a value of type `T`, `&Vec<T>` binds it as `&T`, and `&mut Vec<T>` binds it as `&mut T`. The `Vec` path remains a compiler built-in and does not use the `std::iter::Iterable` fallback.

```lisp
(let mut v (Vec/new))
(Vec/push (& mut v) 1)
(Vec/push (& mut v) 2)
(Vec/push (& mut v) 3)
(for [item (& v)]
  (println! "{}" (* item)))
```

### Dense Flags and Visited Arrays

For a dense flag buffer whose elements are effectively limited to 0 and 1, such as in `nsieve` or a visited/mark array, avoid using `[Vec i32]` or `[Vec bool]` directly. `Vec<bool>` does not compress each element to one bit and is not a replacement for a packed bitset.

The straightforward choice is `[Vec u8]`. When bandwidth and cache efficiency need further optimization, use the `bitset/*` API.

```lisp
(let mut bits (bitset/new-zeroed (to-i64 1000000)))
(bitset/set! (& mut bits) (to-i64 17))
(when (bitset/get (& bits) (to-i64 17))
  (println! "hit"))
(bitset/clear! (& mut bits) (to-i64 17))
(bitset/reset-all! (& mut bits))
```

The public surface consists of the following six functions:

- `bitset/word-count(bit-count)` -> `i64`
- `bitset/new-zeroed(bit-count)` -> `[Vec u64]`
- `bitset/get(bits, idx)` -> `bool`
- `bitset/set!(bits, idx)` -> `()`
- `bitset/clear!(bits, idx)` -> `()`
- `bitset/reset-all!(bits)` -> `()`

`idx` is a bit index and therefore has type `i64`. The internal representation is `Vec<u64>`, but dense-flag use cases should use this surface directly.

### Implicit Heap Layout for Complete Binary Trees

When a payload-free complete binary tree is retained as a long-lived local value and later traversed repeatedly by `check`, `hash`, or `fold`, compare an implicit heap layout before choosing `&Tree` plus an arena. `heaptree/*` is the minimal API for this purpose and treats a `Vec` as the array representation of a complete tree.

```lisp
(let size (heaptree/node-count depth))
(let mut buf : [Vec i32] (Vec/new))
(let mut i 0)
(while (< i size)
  (Vec/push (& mut buf) 1)
  (set! i (+ i 1)))

(defn flat-check [(buf : [& [Vec i32]]) (idx : i32) (depth : i32)] -> i32
  (if (<= depth 0)
    (Vec/get buf idx)
    (+ (Vec/get buf idx)
       (+ (flat-check buf (heaptree/left-index idx) (- depth 1))
          (flat-check buf (heaptree/right-index idx) (- depth 1))))))
```

The public surface consists of the following five functions:

- `heaptree/node-count(depth)` -> `i32`
- `heaptree/leaf-start-index(depth)` -> `i32`
- `heaptree/left-index(idx)` -> `i32`
- `heaptree/right-index(idx)` -> `i32`
- `heaptree/parent-index(idx)` -> `i32`

For a complete binary tree that produces `W-PERF-0053`, evaluate flat indexed traversal with this API first.

---

## Array (Fixed-Length Array)

A fixed-length value type whose size is known at compile time. `Array` itself performs no dynamic heap allocation, but its actual placement depends on the use context and backend, such as whether it is a local value, struct field, or argument.

### Type

```lisp
[Array T N]
```

`T` is the element type, and `N` is the compile-time constant size. For example, `[Array f32 4]` is an array of four `f32` elements.

### Operations

#### `Array/get` -- Element Access

```lisp
(Array/get arr index)
```

Returns the element at the specified index. An out-of-range index **traps (aborts)**. Unlike `Vec/get`, this operation performs a bounds check.

#### `Array/set` -- Element Assignment (Mutating)

```lisp
(Array/set arr index value)
```

Sets the value at the specified index. The array must be declared with `let mut`. The return value is `()`.

```lisp
(let mut arr (array 1 2 3))
(Array/set arr 0 10)
;; arr is (array 10 2 3)
```

#### `Array/len` -- Length

```lisp
(Array/len arr)
```

Returns the length of the array. This value is treated as a compile-time constant.

### Example

```lisp
;; Sum a fixed-length array with a SIMD hint
#[simd :hint :width 4]
(defn sum-array [(arr : [Array f32 16])] -> f32
  (let mut total 0.0f)
  (loop [(i : i32 0)]
    (when (< i 16)
      (set! total (+ total (Array/get arr i)))
      (recur (+ i 1))))
  total)
```

---

## SoaVec (Structure-of-Arrays Dynamic Vector)

`SoaVec` stores each field of a struct type in a separate contiguous buffer. It is a primary target for SIMD gather/scatter optimization.

### Type

```lisp
[SoaVec MyStruct]
```

### Operations

| Function | Description |
|----------|-------------|
| `SoaVec/new` | Creates an empty SoaVec |
| `SoaVec/push` | Appends an element to every field buffer |
| `SoaVec/get-field` | `(SoaVec/get-field (& soa) i field-name)` — reads a field at index `i` |
| `SoaVec/set-field` | `(SoaVec/set-field (& mut soa) i field-name value)` — updates a field |
| `SoaVec/len` | Returns the number of elements as an `i32` |
| `SoaVec/gather` | `(SoaVec/gather (& soa) i)` — explicitly reconstructs one row as a struct |
| `SoaVec/scatter` | `(SoaVec/scatter (& mut soa) i value)` — explicitly updates one row from a struct |

**Note**: `SoaVec/set` (whole-struct update) is not provided (`E-SOA-0001`). Use field-level `SoaVec/set-field` or `SoaVec/scatter`.

```lisp
(defstruct Particle (x : f32) (y : f32) (mass : f32))

(let mut ps : [SoaVec Particle] (SoaVec/new))
(SoaVec/push (& mut ps) (Particle 1.0f 2.0f 0.5f))
(SoaVec/push (& mut ps) (Particle 3.0f 4.0f 1.0f))

;; Field-level access
(let x0 (SoaVec/get-field (& ps) 0 x))  ;; 1.0
(SoaVec/set-field (& mut ps) 0 mass 0.75f)

;; Make the high cost of whole-struct access explicit
(let row (SoaVec/gather (& ps) 1))
```

---

## SoaArray (Fixed-Length Structure-of-Arrays)

A compile-time-sized SoA array `[SoaArray T N]`. There is **no dedicated constructor**; construct one by adding a `[SoaArray T N]` type annotation to an `(array ...)` literal.

```lisp
(defstruct Particle (x : i32) (y : i32))

(let mut ps : [SoaArray Particle 2]
  (array (Particle 1 2) (Particle 3 4)))
```

| Function | Description |
|----------|-------------|
| `SoaArray/get-field` | `(SoaArray/get-field arr i field-name)` — reads a field |
| `SoaArray/set-field` | `(SoaArray/set-field arr i field-name value)` — updates a field |
| `SoaArray/gather` | `(SoaArray/gather arr i)` — reconstructs one row as a struct |
| `SoaArray/scatter` | `(SoaArray/scatter arr i value)` — updates one row from a struct |
| `Array/len` | Returns the number of elements; also applicable to `SoaArray` |

`SoaArray/set` (whole-struct update) is not provided (`E-SOA-0051`).

---

## SoaPVec / SoaPMap (Persistent SoA Collections)

These types combine persistent collections with SoA layout.

```lisp
[SoaPVec T]
[SoaPMap K V]
```

`SoaPVec` is the persistent counterpart of `SoaVec`: `push`, `set-field`, and `scatter` return a new version; `get-field` returns a scalar; `gather` returns a one-row struct; and `len` returns an `i32`. `SoaPMap` provides `new`, `assoc`, `get-field`, `gather`, `contains?`, and `len`.

Fields of `SoaPVec` and keys or values of `SoaPMap` may use any ownable type, including owned values, user-defined aggregates, and nested collections.

---

`HashMap`, `HashSet`, `Heap`, and `Deque` are not built-in types. They are provided as the layer-2 module `std::collections`. Use `std::collections::HashMap` for a fast mutable table, or the persistent data structure `PMap` (`PMap/new`, `PMap/assoc`, `PMap/get`, and so on) when immutability and structural sharing are required.

### Public API

`HashMap K V` is a mutable table with `Hash K` and `Eq K` bounds.

| API | Summary |
|-----|---------|
| `HashMap/new` | Creates an empty map |
| `HashMap/with-capacity` | Creates a map with an initial capacity |
| `HashMap/put` | Inserts a key/value pair and returns the old value as `[Option V]` |
| `HashMap/insert` | Inserts a key/value pair; intended for hot paths that do not use the old value |
| `HashMap/get` | Returns the value for a key as `[Option V]` |
| `HashMap/get-or` | Returns `default` when the key is absent; intended for hot loops that should avoid an `Option` branch |
| `HashMap/contains?` / `HashMap/contains` | Tests whether a key is present |
| `HashMap/remove` | Removes a key and returns the old value as `[Option V]` |
| `HashMap/remove-or` | Removes and returns the value when present, or returns `default` otherwise; intended for hot loops |
| `HashMap/clear` | Removes all elements |
| `HashMap/len` | Returns the number of elements |
| `HashMap/to-vec` / `HashMap/from-vec` | Explicitly converts to and from a logical entry sequence |

`HashSet K` is a thin wrapper around `HashMap K ()` and provides `new`, `with-capacity`, `insert`, `contains?`, `remove`, `clear`, and `len`. `Heap T` is a binary min-heap with an `Ord T` bound and provides `new`, `push`, `pop`, `peek`, `len`, `is-empty`, and `clear`. `Deque T` provides `get`, `push-front`, `push-back`, `pop-front`, `pop-back`, `front`, `back`, `len`, `is-empty`, and `clear`.

### Iterating over `HashMap`

Through the `std::iter::Iterable` fallback, `HashMap` supports iteration with `for [entry (& m)]`. The element type is `[Option [HashMapEntry K V]]`: empty slots and deleted tombstones return `None`, while live entries return `Some(HashMapEntry key value)`. Because the payload vector is append-only, iteration proceeds in insertion order while skipping deleted entries.

```lisp
(use std::collections [HashMap HashMapEntry HashMap/new HashMap/insert])
(let mut m : [HashMap i32 i32] (HashMap/new))
(HashMap/insert (& mut m) 1 10)
(HashMap/insert (& mut m) 2 20)
(for [slot (& m)]
  (match slot
    [None ()]
    [(Some entry) (println! "{}={}" entry.key entry.value)]))
```

`iter-len` and `iter-at` in `std::iter::Iterable` use `i32` indices.

### Serializing `HashMap`

`HashMap` marks its physical layout with `#[serialize-opaque]` and provides a `SerializeAs` implementation that uses `[Vec [HashMapEntry K V]]` as its public wire representation. The stored form is therefore a logical entry sequence rather than a bucket layout, and loading reconstructs the table with `HashMap/from-vec`. When a custom wire struct is required, use `HashMap/to-vec` and `HashMap/from-vec` explicitly.

```lisp
(use std::collections [HashMap HashMap/new HashMap/insert])

(defstruct SaveEntry (key : i32) (value : i32))
(defstruct Save (entries : [Vec SaveEntry]) (tag : u32))
```

---

## Array Combinators

These combinators provide declarative array operations through higher-order functions. The compiler's array-fusion optimization may automatically eliminate intermediate arrays.

### In-Place Transformations

#### `Array/map!` -- In-Place Transformation

```lisp
(Array/map! arr (fn [(x : i32)] -> i32 (* x 2)))
```

Transforms each array element in place. No new array is created.

```lisp
(let mut scores (array 10 20 30))
(Array/map! scores (fn [(x : i32)] -> i32 (* x 2)))
;; scores is (array 20 40 60)
```

#### `Array/zip-map!` -- Pairwise Transformation of Two Arrays

```lisp
(Array/zip-map! a b (fn [(x : i32) (y : i32)] -> i32 (+ x y)))
```

Pairs corresponding elements from two arrays and transforms them. Results are written into the first array.

```lisp
(let mut a (array 1 2 3))
(let b (array 10 20 30))
(Array/zip-map! a b (fn [(x : i32) (y : i32)] -> i32 (+ x y)))
;; a is (array 11 22 33)
```

#### `Array/filter-map!` -- Filter and Transform

```lisp
(Array/filter-map! arr (fn [(x : i32)] -> bool (> x 0)) (fn [(x : i32)] -> i32 (* x 2)))
```

Applies a predicate to each element and transforms only the elements for which it returns `true`. It takes three arguments: the array, predicate, and transformation closure.

### Creating New Arrays

#### `Array/map` -- Create a New Array

```lisp
(Array/map arr (fn [(x : i32)] -> i32 (* x 2)))
```

Returns the transformed result as a new array without modifying the original.

#### `Array/generate` -- Generate from Indices

```lisp
(Array/generate n (fn [(i : i32)] -> i32 (* i i)))
```

Calls the function with each index from 0 through `n-1` and creates an array from the results. The callback's argument type is always `i32`.

```lisp
(let squares (Array/generate 5 (fn [(i : i32)] -> i32 (* i i))))
;; squares is (array 0 1 4 9 16)
```

#### `Array/fill` -- Create a Fixed-Length Array Filled with One Value

```lisp
(Array/fill n value)
```

Because `value` is duplicated into every element, the element type must be `Copy`.

Returns a fixed-size array of length `n` filled with `value`. `n` must be a compile-time integer constant.

```lisp
(let arr : [Array f64 4] (Array/fill 4 2))
;; arr is (array 2.0 2.0 2.0 2.0)
```

### Folds

#### `Array/fold` -- Fold with an Initial Value

```lisp
(Array/fold init arr (fn [(acc : i32) (x : i32)] -> i32 (+ acc x)))
```

Starting from the initial value, accumulates the array elements in order. The arguments are the initial value, array, and closure, in that order.

```lisp
(let total (Array/fold 0 (array 1 2 3 4) (fn [(acc : i32) (x : i32)] -> i32 (+ acc x))))
;; total = 10
```

#### `Array/reduce` -- Associative Fold with an Initial Value

```lisp
(Array/reduce init arr (fn #[associative] [(a : i32) (b : i32)] -> i32 (+ a b)))
```

Starting from the initial value, folds the array with an associative function. It takes the same three arguments as `Array/fold`—the initial value, array, and closure—but the closure must have the `#[associative]` attribute, allowing the compiler to parallelize it with tree reduction.

```lisp
(let total (Array/reduce 0 (array 3 1 4 1 5) (fn #[associative] [(a : i32) (b : i32)] -> i32 (+ a b))))
;; total = 14
```

#### `Array/scan` -- Prefix Fold

```lisp
(Array/scan init arr (fn #[associative] [(acc : i32) (x : i32)] -> i32 (+ acc x)))
```

Accumulates as `Array/fold` does, but returns an array containing each intermediate result. The arguments are the initial value, array, and closure, in that order. The closure must have the `#[associative]` attribute, enabling parallel prefix-scan optimization.

```lisp
(let prefix-sums (Array/scan 0 (array 1 2 3 4) (fn #[associative] [(acc : i32) (x : i32)] -> i32 (+ acc x))))
;; prefix-sums is (array 1 3 6 10)
```

### Array-Fusion Optimization

The compiler automatically fuses consecutive array combinators. For example, consecutive calls to `Array/map!` on the same array are combined into one loop. When the result of `Array/map` is passed directly to `Array/fold`, creation of the intermediate array is eliminated.

```lisp
;; The compiler automatically fuses these into one loop
(Array/map! arr (fn [(x : i32)] -> i32 (* x 2)))
(Array/map! arr (fn [(x : i32)] -> i32 (+ x 1)))

;; Fold directly without an intermediate array
(let sum (Array/fold 0 (Array/map arr (fn [(x : i32)] -> i32 (* x x))) (fn [(acc : i32) (x : i32)] -> i32 (+ acc x))))
```
