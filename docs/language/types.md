# Type System and Built-in Types

Singulisp is statically typed: the type of every value is determined at compile time. HM (Hindley–Milner)-based type inference allows annotations on `defn`, `fn`, and `let` to be omitted where appropriate.

## Overview of Built-in Types

| Category | Types |
|---|---|
| Fixed-width scalars | `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64`, `bool`, `()` |
| Strings and I/O | `String`, `StrSlice`, `Reader`, `Writer` |
| References and functions | `[& T]`, `[& mut T]`, references with lifetimes, `[Fn [...] -> T]` |
| Owned storage | `[Box T]`, `[Vec T]`, `[Array T N]` |
| SoA | `[SoaVec T]`, `[SoaArray T N]` |
| Persistent storage | `[PVec T]`, `[PMap K V]`, `[SoaPVec T]`, `[SoaPMap K V]` |
| Algebraic data | `[Option T]`, `[Result T E]` |
| Low-level SIMD | `f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, `u32x4` |
| Mathematical types | `Vec2`, `Vec3`, `Vec4`, `Quat`, `Float3`, `Mat3`, `Mat3r`, `Mat4`, `Mat4r`, `Mat3x4` |
| GPU and staging | `[GpuBuffer T]`, `[Expr T]` |
| Opaque handles | `ScopeHandle`, `ChannelHandle`, `ModuleHandle`, region handles |
| Built-in errors | `AllocError`, `DeserError` |

`usize` is not a runtime scalar; it is the kind of a const generic parameter. `dyn`, `Rc`, `Arc`, `str`, `char`, `Symbol`, and `Bytes` are not surface types.

### Copy and Move

- Scalars, unit, references, `StrSlice`, low-level SIMD types, mathematical types, and `Expr` are `Copy`.
- `String`, `Box`, `Vec`, SoA and persistent collections, `GpuBuffer`, I/O, concurrency, and region handles, and closures are `Move`.
- Structs, enums, and arrays are `Copy` only when all their components are `Copy`.

---

## Primitive Types

### Integer Types

| Type | Size | Range |
|----|--------|------|
| `i8` | 8 bits | -2^7 to 2^7 - 1 |
| `i16` | 16 bits | -2^15 to 2^15 - 1 |
| `i32` | 32 bits | -2^31 to 2^31 - 1 |
| `i64` | 64 bits | -2^63 to 2^63 - 1 |
| `u8` | 8 bits | 0 to 2^8 - 1 |
| `u16` | 16 bits | 0 to 2^16 - 1 |
| `u32` | 32 bits | 0 to 2^32 - 1 |
| `u64` | 64 bits | 0 to 2^64 - 1 |

```lisp
(let x : i32 42)
```

### Floating-point Types

| Type | Size | Precision |
|----|--------|------|
| `f32` | 32 bits | Single precision (IEEE 754) |
| `f64` | 64 bits | Double precision (IEEE 754) |

```lisp
(let pi : f64 3.14159)
(let half 0.5f)     ; f32
```

### Boolean Type

```lisp
(let flag true)     ; bool
(let done false)
```

The `bool` type has either the value `true` or `false`.

---

## Unit Type

The unit type `()` represents the absence of a value and is used as the return type of an expression whose sole purpose is a side effect.

```lisp
#[external]
(defn greet [(name : String)] -> ()
  (println! "Hello, {}!" name))
```

When the return type is `()`, the `-> ()` annotation may be omitted.

---

## The `String` Type

`String` is an owned, variable-length byte string. It uses SSO to store up to 15 bytes within a 32-byte value; only longer contents own a heap buffer.

```lisp
(let greeting "hello, world")
(let len (String/len greeting))
(let upper (String/to-upper greeting))
```

String-operation functions are provided in the `String/` namespace. Available functions include `String/len`, `String/concat`, `String/substring`, `String/contains`, `String/split`, `String/replace`, `String/to-upper`, and `String/to-lower`.

---

## The `StrSlice` Type

`StrSlice` is a zero-copy, read-only view of string data. It is created by `String/slice`.

```lisp
(let s "hello, world")
(let sl (String/slice (& s) 0 5))     ; StrSlice: "hello"
(String/starts-with sl "hel")          ; => true
```

**Copy semantics**: Because it does not own the buffer, assignment and argument passing perform bitwise copies. Its physical representation is the same 32-byte layout as `String` (`{ ptr, i32, i32, i64, i32, i32 }`), but it does not free the buffer on Drop.

**Escape prohibition (E-IR-0220)**: A `StrSlice` cannot appear in any of the following owning positions. The compiler performs escape checking and reports a violation as a compile-time error.

- function return types;
- struct fields;
- enum-variant payloads; or
- elements of `Vec`, payloads of `Box`, or elements of `Array` (for example, `[Vec StrSlice]`, `[Box StrSlice]`, and `[Array StrSlice N]`).

```lisp
; Examples of compile-time errors
(defstruct Holder
  (s : StrSlice))                       ; E-IR-0220: a StrSlice field is prohibited

(defn bad-slice [(s : String)] -> StrSlice
  (String/slice (& s) 0 3))             ; E-IR-0220: returning StrSlice is prohibited
```

`[& StrSlice]`—access through a reference—is exempt from the escape prohibition.

---

## Reference Types

A reference type does not own a value; it provides indirect access to an existing value.

### Shared References

```lisp
[& T]
```

Multiple shared references may be held simultaneously. They cannot modify the referenced value.

```lisp
(let x 42)
(let r (& x))      ; [& i32]
(println! "{}" (* r))
```

### Mutable References

```lisp
[& mut T]
```

A mutable reference is exclusive. While one exists, no other shared or mutable reference may be created.

```lisp
(let mut x 42)
(let r (& mut x))
(set! (* r) 100)
```

### References with Lifetimes

```lisp
[& ^a T]
```

An explicit lifetime parameter annotates the lifetime of a reference.

```lisp
(defn longest [(a : [& ^l String]) (b : [& ^l String])] -> [& ^l String]
  (if (> (String/len a) (String/len b)) a b))
```

---

## Heap Types

### Box

```lisp
[Box T]
```

Allocates a single value on the heap. Ownership is unique, and the allocation is released automatically at the end of the scope. `Box` is used to define recursive data structures.

```lisp
(let b (Box/new 42))
(println! "{}" (* b))
```

---

## Collection Types

### Vec

```lisp
[Vec T]
```

A variable-length array whose elements can be added and removed. Its internal representation is a 16-byte struct, `{ ptr data, i32 len, i32 cap }`. An arena-backed `[Vec T @region-name]` is a 24-byte struct with an additional back-pointer to the arena.

```lisp
(let mut v (Vec/new))
(Vec/push (& mut v) 1)
(Vec/push (& mut v) 2)
(Vec/push (& mut v) 3)
(println! "{}" (Vec/len (& v)))  ; => 3
```

### SoaVec

```lisp
[SoaVec T]
```

A dynamic array that, when `T` is a struct type, stores each field in a separate buffer (Structure of Arrays). This produces a layout suited to SIMD operations. There is no generic equivalent of `Vec/get`; use `SoaVec/get-field` for field access. Explicitly reconstruct a whole struct with `SoaVec/gather` only where needed.

The physical layout is `{ ptr field0, ptr field1, ..., i32 len, i32 cap }`: one buffer pointer per field, followed by length and capacity. Its size is 8 × the number of fields + 8 bytes.

```lisp
(defstruct Particle (x : f32) (y : f32) (life : f32))

(let mut ps : [SoaVec Particle] (SoaVec/new))
(SoaVec/get-field (& ps) 0 x)   ; retrieve only field x → f32
```

### Array

```lisp
[Array T N]
```

A fixed-length array. Its size `N` must be a compile-time constant.

```lisp
(let arr (array 1 2 3 4 5))  ; [Array i32 5]
```

---

## Function Types

A function type consists of a list of parameter types and a return type.

```lisp
[Fn [i32 i32] -> i32]
```

It is used as the parameter type of a higher-order function or when binding a function to a variable.

```lisp
(defn apply [(f : [Fn [i32] -> i32]) (x : i32)] -> i32
  (f x))

(let double (fn [(x : i32)] -> i32 (* x 2)))
(apply double 5)  ; => 10
```

---

## The `Option` Type

```lisp
[Option T]
```

A built-in type representing the presence or absence of a value. It has two variants, `Some` and `None`.

```lisp
(let found (Some 42))       ; [Option i32]
(let missing None)           ; [Option T] (through type inference)

(match found
  [(Some x) (println! "found: {}" x)]
  [None     (println! "not found")])
```

---

## The `Result` Type

```lisp
[Result T E]
```

A built-in type representing a success or failure value. It has two variants, `Ok` and `Err`.

```lisp
(defn divide [(a : f64) (b : f64)] -> [Result f64 String]
  (if (== b 0.0)
    (Err "division by zero")
    (Ok (/ a b))))
```

### The `?` Operator

The `?` operator provides concise error propagation for `Result`.

```lisp
(defn compute [(x : f64)] -> [Result f64 String]
  (let a (? (divide x 2.0)))
  (let b (? (divide a 3.0)))
  (Ok b))
```

If the expression is `Err`, `?` immediately returns from the function. If it is `Ok`, `?` extracts the contained value.

---

## SIMD Types

Singulisp's SIMD support consists of two families.

- **Low-level SIMD types**: `IrType::SimdVec` types such as `f32x4`, `f32x2`, and `i32x4`. They map directly to SIMD registers and are used for low-level optimization, SPMD blocks, and bitwise operations.
- **High-level mathematical types**: `Vec2`, `Vec3`, `Vec4`, `Quat`, `Mat3`, `Mat4`, and others. They are retained as dedicated IR, and the backend lowers them according to the operation to vector instructions, LLVM intrinsics, libm, or fast kernels. See the next section.

### Low-level SIMD Types (SimdVec)

All have Copy semantics. In LLVM they are `<N x ElemType>` vector types.

| Type | Element type | Elements | Bytes |
|----|--------|--------|---------|
| `f32x2` | f32 | 2 | 8 |
| `f32x4` | f32 | 4 | 16 |
| `f64x2` | f64 | 2 | 16 |
| `i32x2` | i32 | 2 | 8 |
| `i32x4` | i32 | 4 | 16 |
| `u32x4` | u32 | 4 | 16 |

```lisp
(let a (f32x4 1.0f 2.0f 3.0f 4.0f))
(let b (f32x4 5.0f 6.0f 7.0f 8.0f))
(let c (simd/add a b))  ; element-wise addition
```

Principal built-in operations include `simd/add`, `simd/sub`, `simd/mul`, `simd/div`, `simd/dot`, `simd/extract`, `simd/insert`, and `simd/splat`.

---

## Mathematical Types (High-level Built-in SIMD Types)

These high-level mathematical types are represented as `IrType::Math(MathType)`. Unlike a `:repr simd` struct, they have dedicated IR types, and the backend lowers each operation through the appropriate path. All have Copy semantics, but they are not guaranteed always to reside in physical registers.

These type names are reserved and cannot be redefined as user-defined structs or enums (`E-IR-0150`). They receive the same treatment as other built-in type names such as `String`, `Vec`, and `Option`.

### Vector, Quaternion, and Packed Types

| Type | LLVM type | Bytes | Notes |
|------|---------|---------|------|
| `Vec2` | `<2 x float>` | 8 | 2D vector |
| `Vec3` | `<4 x float>` | 16 | 3D vector (unused w lane, 16B aligned) |
| `Vec4` | `<4 x float>` | 16 | 4D vector |
| `Quat` | `<4 x float>` | 16 | Quaternion (XYZW order) |
| `Float3` | `{f32, f32, f32}` packed | 12 | 3D vector for array storage; non-SIMD |

`Vec3` is processed in SIMD registers, but its w lane is unused and has an indeterminate value. `Float3` is packed into 12 bytes and is suitable for dense array storage such as `Vec<Float3>`. Convert between them with `(Vec3/of-float3 f)` and `(Float3/of-vec3 v)`.

### Matrix Types

| Type | Bytes | Storage convention |
|------|---------|-------------|
| `Mat3` | 48 | 3×3 column-major |
| `Mat3r` | 48 | 3×3 row-major |
| `Mat4` | 64 | 4×4 column-major |
| `Mat4r` | 64 | 4×4 row-major |
| `Mat3x4` | 48 | 3-row × 4-column row-major affine (bottom row `(0,0,0,1)` omitted) |

`Mat3` and `Mat4` support `inverse` and `determinant`; `Mat3x4` supports affine `inverse`. No same-type transpose or determinant is defined for `Mat3x4`. GPU convention: WGSL/SPIR-V uses column-major (`Mat3`/`Mat4`), while HLSL uses row-major (`Mat3r`/`Mat4r`).

### Principal Operations

```lisp
(let v (Vec3 1.0f 2.0f 3.0f))        ; construction
(let s (Vec3/splat 1.0f))              ; same value in every component
(let d (Vec3/dot v v))                 ; dot product (f32)
(let n (Vec3/normalize v))             ; normalization
(let l (Vec3/lerp v s 0.5f))           ; linear interpolation

(let m (Mat4/identity))
(let v4 (Vec4 1.0f 2.0f 3.0f 1.0f))
(let r (Mat4/transform m v4))          ; 4×4 matrix × Vec4
(let q (Quat/slerp q1 q2 0.5f))       ; spherical interpolation
```

For details, see [First-class SIMD Mathematical Types](math-types.md).

---

## GPU Types

### GpuBuffer

```lisp
[GpuBuffer T]
```

A typed array in GPU device memory. On the CPU side, it is represented as a 24-byte struct, `{ ptr data, i32 len, i32 elem_size, i32 mode }`. Within a GPU kernel, it maps to a WGSL `array<T>` storage buffer. It has Move semantics and is used as a parameter of a `#[shader]` function.

Element type `T` is limited to Copy and GPU POD types. Supported types are `i32`, `u32`, and `f32`; two- to four-lane SIMD types containing those lane types; mathematical types other than `Float3`; fixed-length `[Array T N]` of supported types; and `:repr gpu` structs whose every field recursively satisfies these conditions. `bool`, 64-bit scalars, `String`, dynamic collections, references, enums, and `Float3` cannot be element types.

```lisp
(defstruct Particle :repr gpu
  (position : Vec3)
  (velocity : Vec3))

#[shader :thread-group-size [256 1 1]]
(defn update [(particles : [& mut [GpuBuffer Particle]]) (dt : f32)] -> ()
  ...)
```

---

## Persistent Data-structure Types

Persistent (immutable) data structures supporting a functional programming style.

### PVec

```lisp
[PVec T]
```

A persistent vector. An update operation returns a new version without modifying the original data. Internally it is implemented as a Clojure-style 32-way persistent vector trie with a tail.

`T` may be any ownable type, including owned values, user-defined aggregates, and nested collections.

```lisp
(let v1 (PVec/new))
(let v2 (PVec/push v1 10))
(let v3 (PVec/push v2 20))
; v1 remains empty, while v3 is [10, 20]
```

### PMap

```lisp
[PMap K V]
```

A persistent map: an immutable hash map storing key-value pairs. It is implemented with a HAMT and bitmap compression; update operations return a new map through structural sharing.

Keys and values may be any ownable types, including owned values, user-defined aggregates, and nested collections.

**`PMap/get` behavior**: Returns the corresponding value `V` directly, not `[Option V]`. For a missing key, the current runtime returns the zero value of the value representation. To distinguish absence from a zero value, first check with `PMap/contains?`.

```lisp
(let m1 (PMap/new))
(let m2 (PMap/assoc m1 1 100))
(let m3 (PMap/assoc m2 2 200))
(PMap/get m3 1)  ; => 100 (returns V directly)
```

---

## Channel

An MPMC (multi-producer, multi-consumer) channel for inter-thread communication. Internally it is implemented as a mutex-and-condition-variable-based ring buffer.

Create a channel with `channel/new` and operate on it with `channel/send`, `channel/recv`, and `channel/close`. Its surface type is the non-generic `ChannelHandle`, and payloads are limited to `i32`. It may be explicitly annotated as `ChannelHandle` when needed.

```lisp
(let ch (channel/new 16))   ; buffer size 16
(channel/send ch 42)
(let val (channel/recv ch)) ; => 42
(channel/close ch)
```

---

## Type Inference

Singulisp uses HM-based type inference. Parameter and return types on `defn` and `fn`, as well as type annotations on `let`, may be omitted and written only where needed.

```lisp
(defn add [a b] -> i32
  (+ a b))

(let double (fn [x] (* x 2)))
```

When context determines the width of an integer literal, that type is inferred. Otherwise, a default is applied: `i32` when the value fits in the `i32` range, then `i64`, and finally `u64`. An unsuffixed floating-point literal is `f64`, while one with an `f` suffix is `f32`.

```lisp
; OK: defaults to i32 when context does not determine a width
(defn defaulted []
  (+ 1 2))         ; i32

; OK: the return type explicitly determines the width
(defn fixed [] -> i64
  (+ 1 2))         ; i64

; OK: an unsuffixed floating-point literal is f64
(defn ratio [] -> f64
  3.14)
```

---

## Type Casts

The built-in `to-*` functions perform explicit type conversions.

| Function | Target type |
|------|-----------|
| `to-i8` | i8 |
| `to-i16` | i16 |
| `to-i32` | i32 |
| `to-i64` | i64 |
| `to-u8` | u8 |
| `to-u16` | u16 |
| `to-u32` | u32 |
| `to-u64` | u64 |
| `to-f32` | f32 |
| `to-f64` | f64 |

```lisp
(to-u8 255)        ; i32 → u8
(to-i64 42)        ; i32 → i64
(to-f32 3.14)      ; f64 → f32
(to-i32 true)      ; bool → i32 (1)
```

Casts between numeric types permit narrowing conversions to a lower-precision type, but values may be truncated. Integer narrowing preserves the low-order bits. Floating-point-to-integer conversion rounds toward zero and then saturates to the target integer's range (**saturating conversion**). NaN becomes 0, as does a negative value converted to an unsigned integer.

```lisp
(to-i32 3.9)      ; => 3 (rounded toward zero)
(to-i32 1e18)     ; => 2147483647 (clamped to i32::MAX)
(to-i32 f32/NAN)  ; => 0
```

Safe implicit widening conversions are also permitted. For example, `u8 -> u32`, `i8 -> i16`, and `f32 -> f64` may be implicit. Dangerous conversions involving narrowing or sign changes, as well as mixtures with no common type, produce `E-TYPE-1007`.

### Integer Shift Operations

The shift amount of `bit-shl` and `bit-shr` is masked by the type width, so an amount greater than or equal to the width does not cause undefined behavior.

- Variable shift amount: automatically inserts `amount AND (type width - 1)` (for example, AND 63 for `i64`).
- In-range constant shift amount: omits the mask as an optimization.
- Out-of-range constant shift amount: constant-folds the same mask based on the type width.

```lisp
(bit-shl x n)    ; variable n → automatically insert AND 63 for i64
(bit-shl x 3)    ; constant 3 is in range → omit the mask
(bit-shl x 64)   ; for i64, 64 AND 63 = 0, so the result is x
```

---

## `repr` Specifications

A `repr` specification on a struct definition controls memory layout.

| repr | Effect |
|------|------|
| `:repr C` | C-compatible memory layout for FFI |
| `:repr simd` | Hint requiring 16-byte alignment at the corresponding stack or arena allocation site; does not change type size, ABI alignment, or array stride |
| `:repr gpu` | std430 layout for GPU-buffer compatibility |
| `:repr packed` | Removes padding in declaration order and handles unaligned field access |
| `:repr ordered` | Preserves field declaration order |

```lisp
(defstruct AlignedPoint3 :repr simd
  (x : f32) (y : f32) (z : f32))

(defstruct Vertex :repr gpu
  (position : Vec3)
  (normal   : Vec3)
  (uv       : f32x2))

(defstruct Header :repr C
  (magic   : u32)
  (version : u32)
  (flags   : u32))
```

---

## `deftype-fn` (Type-level Functions)

`deftype-fn` defines type-level computation. It accepts type parameters and returns a type.

```lisp
;; Example with a const parameter: a fixed-length buffer of N elements of type T
(deftype-fn FixedBuffer [(N : usize) (T : Type)] :repr (Array T N))
```

A type-level function is used to define a generic type alias or parametric type. Parameters use the form `(name : kind)`, where `kind` is `Type` for a type parameter or `usize` for a const parameter. Write the expansion type form after `:repr`.

```lisp
;; Example with only a type parameter: an alias for a vector of T
(deftype-fn VecOf [(T : Type)] :repr (Vec T))
```

**`usize` restriction**: `usize` exists only for const generic parameters and is not a runtime value type. There is no `to-usize` cast function. To use it as a value, convert it to an ordinary integer type with `(to-i32 N)` or a similar function. In IR it maps to `i32` for values below 2³¹ and otherwise to `i64`.

**Expansion constraints**:

| Error code | Condition |
|---|---|
| `E-TYPE-0501` | Expansion is cyclic through direct or indirect recursion |
| `E-TYPE-0502` | Expansion depth exceeds the limit of 256 levels |
