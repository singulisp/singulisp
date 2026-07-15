# SIMD

Singulisp's SIMD support consists of two systems.

- **Low-level SIMD types** (`f32x4` and others, with `simd/*` operations): used for portable bit operations, internal representations within SPMD blocks, and low-level optimization
- **High-level mathematical types** (`Vec2`, `Vec3`, `Vec4`, `Quat`, `Mat3`, `Mat4`, and others): dedicated IR types for numeric computation, lowered by the backend to vector instructions, LLVM intrinsics, libm, or fast kernels according to the operation

Use high-level mathematical types when type-specific meaning matters; use low-level SIMD types when explicit lane operations or masks are required.

## Low-level SIMD types

They are represented as `IrType::SimdVec(elem_type, lanes)`.

| Type | Elements | LLVM type | Description |
|---|---|---|---|
| `f32x4` | 4 x f32 | `<4 x float>` | Four single-precision floating-point elements |
| `f32x2` | 2 x f32 | `<2 x float>` | Two single-precision floating-point elements |
| `f64x2` | 2 x f64 | `<2 x double>` | Two double-precision floating-point elements |
| `i32x4` | 4 x i32 | `<4 x i32>` | Four signed integer elements |
| `i32x2` | 2 x i32 | `<2 x i32>` | Two signed integer elements |
| `u32x4` | 4 x u32 | `<4 x i32>` | Four unsigned integer elements |

This table is the complete list of the six named types in the surface language. The optimizer or
`simd/splat` may create other lane counts internally, but there is no named `f32x8` type or constructor.

## Literals and construction

SIMD values are created with constructor forms.

```lisp
(let a (f32x4 1.0f 2.0f 3.0f 4.0f))
(let b (f32x2 1.0f 2.0f))
(let c (f64x2 1.0 2.0))
(let d (i32x4 1 2 3 4))
(let e (i32x2 1 2))
(let f (u32x4 1 2 3 4))
```

## Arithmetic operations

Use functions in the `simd/` namespace to operate on low-level SIMD types. Ordinary arithmetic operators (`+`, `-`, `*`, `/`) cannot be used with SIMD types.

```lisp
(let a (f32x4 1.0f 2.0f 3.0f 4.0f))
(let b (f32x4 5.0f 6.0f 7.0f 8.0f))
(let c (simd/add a b))  ; (f32x4 6.0f 8.0f 10.0f 12.0f)
(let d (simd/mul a b))  ; (f32x4 5.0f 12.0f 21.0f 32.0f)
```

| Function | Description |
|---|---|
| `simd/add` | Element-wise addition |
| `simd/sub` | Element-wise subtraction |
| `simd/mul` | Element-wise multiplication |
| `simd/div` | Element-wise division |
| `simd/min`, `simd/max` | Element-wise minimum and maximum |
| `simd/lt`, `gt`, `le`, `ge`, `eq`, `ne` | Element-wise comparisons; return an unnamed Boolean SIMD mask |
| `simd/select` | Masked element selection |
| `simd/shuffle` | Lane rearrangement |
| `simd/sqrt`, `abs`, `floor`, `ceil`, `round`, `trunc`, `fract`, `sign`, `rsqrt` | Element-wise unary operations |
| `simd/sin`, `cos`, `tan`, `exp`, `exp2`, `ln`, `log2`, `log10` | Element-wise mathematical functions |

`simd/abs` supports every numeric lane type. The other unary mathematical functions in the table are
limited to `f32` and `f64` lanes and cannot be used with integer SIMD types.

## SIMD operations

### Dot product

```lisp
(simd/dot a b)
```

Computes the dot product of two SIMD vectors and returns a scalar value.

### Element access

```lisp
(simd/extract v index)       ; Get the element at index
(simd/insert v index value)  ; Return a new vector with the element at index replaced
```

`index` must be a compile-time constant.

### Splat

```lisp
(simd/splat value lane-count)  ; Initialize every element to the same value
```

```lisp
(let ones (simd/splat 1.0f 4))  ; (f32x4 1.0f 1.0f 1.0f 1.0f)
```

`lane-count` is also a compile-time constant.

## `:repr simd` — SIMD alignment hint

On a struct, `:repr simd` is a layout hint that requests 16-byte alignment at corresponding allocation
sites. The current LLVM backend does not add tail padding to struct types, so this does not guarantee
that the type's size or array stride is rounded up to 16 bytes.

```lisp
(defstruct AlignedPoint3 :repr simd
  (x : f32) (y : f32) (z : f32))
```

The type size and array stride of this three-`f32` struct remain 12 bytes, and 16-byte vector loads and
stores are not guaranteed. For a stable 16-byte layout, add an explicit fourth field or use
`:repr aligned 16`, then verify it with `gu-cli asm` and ABI output.

## `#[simd :hint :width N]` — vectorization-width hint

This hint communicates a preferred width to analyses and diagnostics.

```lisp
#[simd :hint :width 8]
(defn process-array [(data : [Array f32 1024])] -> [Array f32 1024]
  (Array/map data (fn [(x : f32)] -> f32 (* x 2.0f))))
```

The attribute value is used by SPMD- and window-related analyses and diagnostics but does not control
the LLVM loop vectorizer's VF. It therefore does not guarantee instructions of width `N`. The target,
loop shape, and optimizer determine the actual width; inspect it with `gu-cli asm`.

## Collection-side SoA

Singulisp represents SoA with collection types rather than struct attributes. What matters to SIMD optimization is not `Particle` as a value type, but how `Particle` values are arranged.

```lisp
(defstruct Particle
  (x : f32) (y : f32) (z : f32) (life : f32))

(let ps : [SoaArray Particle 1024] (array ...))
(let x0 (SoaArray/get-field ps 0 x))
```

Ordinary AoS layout:
```
[x0,y0,z0,life0, x1,y1,z1,life1, ...]
```

SoA layout:
```
[x0,x1,...][y0,y1,...][z0,z1,...][life0,life1,...]
```

In SoA, values of the same field are contiguous in memory, making bulk SIMD processing efficient. Use `SoaArray/gather` or `SoaVec/gather` explicitly only when an entire struct must be read.

## Practical example: SIMD-optimized dot product

This example expresses the dot product of a three-component struct with a low-level SIMD type.

```lisp
(defstruct Point3 :repr simd
  (x : f32) (y : f32) (z : f32))

(defn point3-dot-simd [(a : Point3) (b : Point3)] -> f32
  (let va : f32x4 (f32x4 a.x a.y a.z 0.0f))
  (let vb : f32x4 (f32x4 b.x b.y b.z 0.0f))
  (simd/dot va vb))
```

The fourth element is padded with `0.0f`, so it does not affect the result. Final instruction selection
depends on the target and optimization mode and can be inspected with `gu-cli asm`.

## Floating-point optimization attributes

These floating-point optimization attributes are often used with SIMD code.

```lisp
;; Permit FMA (Fused Multiply-Add)
#[fp-mode :contract]
(defn fma-example [(a : f32x4) (b : f32x4) (c : f32x4)] -> f32x4
  (simd/add (simd/mul a b) c))  ; a*b+c may become a single FMA instruction

;; Full fast-math mode
#[fp-mode :fast]
(defn fast-magnitude [(v : Vec3)] -> f32
  (f32/sqrt (+ (+ (* v.x v.x) (* v.y v.y)) (* v.z v.z))))
```
