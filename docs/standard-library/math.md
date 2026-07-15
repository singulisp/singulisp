# Math Functions

Singulisp provides built-in math functions for numerical computing. Depending on the
target and mode, they are lowered to LLVM intrinsics or to math kernels within the
implementation.

---

## Floating-Point Constants

The following constants are available in both the `f32/` and `f64/` namespaces.
**They are zero-argument constants**; passing an argument produces `E-IR-0050`.

| Constant | Description | Approximate f32 value |
|------|------|-----------|
| `f32/INF` / `f64/INF` | Positive infinity | `+∞` |
| `f32/NAN` / `f64/NAN` | Not a number (NaN) | `NaN` |
| `f32/MAX` / `f64/MAX` | Largest finite value | `3.4028235e38` |
| `f32/MIN` / `f64/MIN` | **Most negative finite value** (≈ −3.4028235e38) | `-3.4028235e38` |
| `f32/EPSILON` / `f64/EPSILON` | Machine epsilon (smallest difference from 1.0) | `1.1920929e-7` |
| `f32/PI` / `f64/PI` | The constant π | `3.1415927` |
| `f32/TAU` / `f64/TAU` | 2π | `6.2831855` |
| `f32/E` / `f64/E` | Base of the natural logarithm, e | `2.7182817` |

**Note:** `f32/MIN` is the **most negative finite value**, equivalent to Rust's
`f32::MIN`. It is not GLSL's `FLT_MIN`, which is the smallest positive normal value
(approximately 1.175494e-38).

---

## Unary Functions

Every function below is available in both the `f32/` and `f64/` namespaces. The tables
show the `f32/` variant.

### Basic Algebra and Trigonometry

| Function | Description | Domain / range |
|------|------|--------------|
| `f32/sqrt` | Square root | x ≥ 0 (negative values → NaN) |
| `f32/cbrt` | Cube root | All values |
| `f32/abs` | Absolute value | All values |
| `f32/sign` | Sign (-1.0, 0.0, 1.0) | `{−1, 0, +1}` |
| `f32/recip` | Reciprocal (1/x) | x = 0 → ±Inf |
| `f32/rsqrt` | `1 / sqrt(x)`; fast mode permits approximate optimization | x > 0 |
| `f32/sin` | Sine (radians) | [−1, 1] |
| `f32/cos` | Cosine (radians) | [−1, 1] |
| `f32/tan` | Tangent (radians) | All values |
| `f32/asin` | Inverse sine | −1 ≤ x ≤ 1 → [−π/2, π/2] |
| `f32/acos` | Inverse cosine | −1 ≤ x ≤ 1 → [0, π] |
| `f32/atan` | Inverse tangent (one argument) | → [−π/2, π/2] |

### Hyperbolic Functions

| Function | Description |
|------|------|
| `f32/sinh` | Hyperbolic sine |
| `f32/cosh` | Hyperbolic cosine |
| `f32/tanh` | Hyperbolic tangent |
| `f32/asinh` | Inverse hyperbolic sine |
| `f32/acosh` | Inverse hyperbolic cosine |
| `f32/atanh` | Inverse hyperbolic tangent |

### Exponentials and Logarithms

| Function | Description |
|------|------|
| `f32/exp` | e^x |
| `f32/exp2` | 2^x |
| `f32/exp10` | 10^x |
| `f32/expm1` | e^x − 1, preserving precision near x = 0 |
| `f32/ln` | Natural logarithm |
| `f32/log2` | Base-2 logarithm |
| `f32/log10` | Base-10 logarithm |
| `f32/log1p` | ln(1+x), preserving precision near x = 0 |

### Rounding and Fractional Parts

| Function | Description |
|------|------|
| `f32/floor` | Floor (toward −∞) |
| `f32/ceil` | Ceiling (toward +∞) |
| `f32/round` | Round to nearest |
| `f32/trunc` | Truncate toward zero |
| `f32/fract` | Fractional part (x − floor(x)) |

### Unary Numerical Utilities

| Function | Description |
|------|------|
| `f32/saturate` | Clamp to [0, 1] |
| `f32/degrees` | Radians → degrees |
| `f32/radians` | Degrees → radians |

### Predicates (Return Type `bool`)

| Function | Description |
|------|------|
| `f32/isnan` | Whether the value is NaN |
| `f32/isinf` | Whether the value is ±infinity |
| `f32/isfinite` | Whether the value is finite (`false` for NaN or infinity) |

### Example

```lisp
(let x 2.0f)
(let root (f32/sqrt x))          ;; 1.4142135
(let angle 1.5707963f)
(let s (f32/sin angle))          ;; 1.0 (sin(pi/2))
(let rounded (f32/floor 3.7f))   ;; 3.0
```

---

## Multi-Argument Functions

Every function below is available in both the `f32/` and `f64/` namespaces.

### Two-Argument Functions

| Function | Arguments | Description |
|------|------|------|
| `f32/min` | a b | Minimum value (does not propagate NaN) |
| `f32/max` | a b | Maximum value (does not propagate NaN) |
| `f32/pow` | base exp | Exponentiation, base^exp |
| `f32/atan2` | y x | Four-quadrant inverse tangent → (−π, π] |
| `f32/copysign` | mag sign | Magnitude of `mag` with the sign of `sign` |
| `f32/hypot` | x y | sqrt(x²+y²), avoiding overflow |
| `f32/step` | edge x | x < edge → 0.0; x ≥ edge → 1.0 |
| `f32/fmod` | x y | Floating-point remainder, identical to C `fmod` |
| `f32/repeat` | t length | Wraps within [0, length) |
| `f32/wrap-signed` | t length | Wraps within [−length/2, length/2) |
| `f32/ping-pong` | t length | Oscillates 0 → length → 0 |
| `f32/delta-angle` | current target | Shortest-path angular difference |

**Note:** `f32/step` takes arguments as `(f32/step edge x)`, with `edge` first, as in
GLSL.

### Three-Argument Functions

| Function | Arguments | Description |
|------|------|------|
| `f32/fma` | a b c | Fused multiply-add a×b+c (`llvm.fma` intrinsic) |
| `f32/lerp` | a b t | Linear interpolation a+(b-a)×t (`f32/mix` is equivalent) |
| `f32/mix` | a b t | Alias of `lerp` |
| `f32/clamp` | x lo hi | Clamp to [lo, hi] |
| `f32/smoothstep` | edge0 edge1 x | Cubic Hermite interpolation → [0, 1] |
| `f32/smootherstep` | edge0 edge1 x | Quintic interpolation with greater smoothness → [0, 1] |
| `f32/inverse-lerp` | a b value | Inverse lerp (value → t) |
| `f32/lerp-angle` | a b t | Shortest-path angular interpolation (radians) |

### Five-Argument Functions

| Function | Arguments | Description |
|------|------|------|
| `f32/remap` | value from-min from-max to-min to-max | Remaps [from-min, from-max] to [to-min, to-max] |

### Example

```lisp
(let a 3.0f)
(let b 7.0f)
(let smaller (f32/min a b))       ;; 3.0
(let larger (f32/max a b))        ;; 7.0
(let squared (f32/pow a 2.0f))    ;; 9.0
(let angle (f32/atan2 1.0f 1.0f)) ;; 0.7853982 (pi/4)
```

---

## Floating-Point-to-Integer Conversion

Floating-point-to-integer conversions such as `to-i32` and `to-i64` are
**saturating conversions**. NaN becomes 0, and out-of-range values saturate to the
target type's MIN or MAX; no undefined behavior occurs. See the type-conversion
section of `utilities.md` for details.

---

## Short Forms

Short forms that omit the namespace prefix are available. The argument types select
the appropriate f32 or f64 variant; these forms do not default to f64.

| Short form | Description |
|--------|------|
| `sqrt` | `f32/sqrt` or `f64/sqrt`, according to the argument type |
| `abs` | Floating-point or any integer-width variant, according to the argument type |
| `sin` | `f32/sin` or `f64/sin`, according to the argument type |
| `cos` | `f32/cos` or `f64/cos`, according to the argument type |
| `tan` | `f32/tan` or `f64/tan`, according to the argument type |
| `asin` | `f32/asin` or `f64/asin`, according to the argument type |
| `acos` | `f32/acos` or `f64/acos`, according to the argument type |
| `floor` | `f32/floor` or `f64/floor`, according to the argument type |
| `ceil` | `f32/ceil` or `f64/ceil`, according to the argument type |
| `min` | Floating-point or any integer-width variant, according to the argument type |
| `max` | Floating-point or any integer-width variant, according to the argument type |

```lisp
;; Short forms inferred from argument types
(let x (sqrt 2.0))      ;; f64 argument → f64/sqrt
(let y (sqrt 2.0f))     ;; f32 argument → f32/sqrt
(let z (abs -5))        ;; i32 argument → i32 abs variant
(let p (f64/pow 2.0 3.0))   ;; Use namespace form for binary functions

;; Explicit namespace variants
(let x (f32/sqrt 2.0f))
(let p (f64/pow 2.0 3.0))
```

---

## SIMD Math

Math operations on SIMD types execute vector operations in parallel at the hardware
level.

### Arithmetic Operations

The six named SIMD types are `f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, and
`u32x4`. Basic operations use dedicated functions in the `simd/` namespace. Standard
arithmetic operators (`+`, `-`, `*`, `/`) cannot be applied to SIMD types.

| Function | Description |
|------|------|
| `simd/add` | Lane-wise addition |
| `simd/sub` | Lane-wise subtraction |
| `simd/mul` | Lane-wise multiplication |
| `simd/div` | Lane-wise division |
| `simd/min` / `simd/max` | Lane-wise minimum / maximum |
| `simd/extract` / `simd/insert` | Read / update a lane |
| `simd/shuffle` | Reorder lanes |
| `simd/lt` and related functions | Lane-wise comparison mask |
| `simd/select` | Select lanes with a mask |
| `simd/splat` | Replicate a scalar to the specified lane count |
| `simd/sqrt`, `simd/abs` | Square root / absolute value |
| `simd/floor`, `simd/ceil`, `simd/round`, `simd/trunc` | Lane-wise rounding |
| `simd/fract`, `simd/sign`, `simd/rsqrt` | Fractional part / sign / reciprocal square root |
| `simd/sin`, `simd/cos`, `simd/tan` | Lane-wise trigonometric functions |
| `simd/exp`, `simd/exp2`, `simd/ln`, `simd/log2`, `simd/log10` | Lane-wise exponential and logarithmic functions |

`simd/abs` accepts every numeric lane type. The other unary math functions accept
only `f32` or `f64` lanes and cannot be used with integer SIMD types.

```lisp
(let a : f32x4 (f32x4 1.0f 2.0f 3.0f 4.0f))
(let b : f32x4 (f32x4 5.0f 6.0f 7.0f 8.0f))
(let c (simd/add a b))  ;; (6.0 8.0 10.0 12.0)
(let d (simd/mul a b))  ;; (5.0 12.0 21.0 32.0)
```

### `simd/dot` — Dot Product

```lisp
(simd/dot a b)
```

Returns the dot product of two vectors with the same lane type and count, using that
lane's scalar type as the result type.

```lisp
(let a : f32x4 (f32x4 1.0f 2.0f 3.0f 4.0f))
(let b : f32x4 (f32x4 5.0f 6.0f 7.0f 8.0f))
(let dot (simd/dot a b))  ;; 70.0 (= 1*5 + 2*6 + 3*7 + 4*8)
```

---

## Floating-Point Modes

The `#[fp-mode]` attribute controls the optimization level of floating-point
operations.

### `#[fp-mode :contract]` — Permit FMA

Permits fused multiply-add, improving performance while preserving precision.

```lisp
#[fp-mode :contract]
(defn dot-product [(a : Vec3) (b : Vec3)] -> f32
  (+ (* a.x b.x) (+ (* a.y b.y) (* a.z b.z))))
```

### `#[fp-mode :fast]` — Fast Floating Point

Relaxes strict IEEE 754 conformance and permits maximum optimization, including the
application of associativity and distributivity, constant folding, and reciprocal
approximations.

```lisp
#[fp-mode :fast]
(defn approx-distance [(dx : f32) (dy : f32)] -> f32
  (f32/sqrt (+ (* dx dx) (* dy dy))))
```

### `#[deterministic]` — Deterministic Mode

`#[deterministic]` uses built-in math kernels for `sin`, `cos`, `exp`, `ln`, and
related functions without applying fast-math flags. Falling back to libm or LLVM
intrinsics would introduce platform differences, so combining this attribute with
`#[opt :math-kernel-expand false]` is an error. `spmd-vectorize` is also disabled by
default to avoid differences in floating-point reduction order across SIMD widths.

The kernels use `llvm.fma.*` and require IEEE FMA. Code generation is rejected with
`E-DET-0002` when FMA cannot be established through `+fma`, a known CPU baseline that
includes FMA, or the AArch64 baseline.

---

## Numerical Examples

### Vector Length (Magnitude)

```lisp
(defn length [(v : Vec3)] -> f32
  (f32/sqrt (+ (* v.x v.x) (+ (* v.y v.y) (* v.z v.z)))))
```

### Vector Normalization

```lisp
(defn normalize [(v : Vec3)] -> Vec3
  (let len (length v))
  (Vec3 (/ v.x len) (/ v.y len) (/ v.z len)))
```

### Linear Interpolation (lerp)

```lisp
(defn lerp [(a : f32) (b : f32) (t : f32)] -> f32
  (+ (* a (- 1.0f t)) (* b t)))
```

### Angle Conversion

```lisp
(defn deg-to-rad [(deg : f32)] -> f32
  (* deg (/ 3.14159265f 180.0f)))

(defn rad-to-deg [(rad : f32)] -> f32
  (* rad (/ 180.0f 3.14159265f)))
```

### Clamping a Value

```lisp
(defn clamp [(x : f32) (lo : f32) (hi : f32)] -> f32
  (f32/min (f32/max x lo) hi))
```

### 2D Rotation

```lisp
(defn rotate-2d-x [(x : f32) (y : f32) (angle : f32)] -> f32
  (let c (f32/cos angle))
  (let s (f32/sin angle))
  (- (* x c) (* y s)))
```

---

## `std::math`

`std::math` provides inline helpers composed from built-in operations.

```lisp
(use std::math :as math)
```

| Category | Public APIs |
|---|---|
| Clamp / interpolation | `math/clamp01`, `clamp`, `lerp`, `inv-lerp`, `remap`, `smoothstep`, `smootherstep` |
| Vector interpolation | `math/lerp-v2`, `math/lerp-v3` |
| Angles | `math/deg-to-rad`, `rad-to-deg`, `wrap-angle`, `lerp-angle` |
| Easing | `math/ease-in-quad`, `ease-out-quad`, `ease-in-out-quad`, `ease-in-cubic`, `ease-out-cubic`, `ease-in-out-cubic`, `ease-out-elastic`, `ease-out-bounce` |
| Scalar Bézier | `math/bezier-quad`, `math/bezier-cubic` |
| Vector Bézier | `math/bezier-quad-v2`, `bezier-cubic-v2`, `bezier-quad-v3`, `bezier-cubic-v3` |
| Deterministic noise | `math/value-noise-1d`, `math/perlin-2d` |

Layer-2 helpers primarily target `f32` and the built-in `Vec2` and `Vec3` types. Use
compiler built-in APIs for a wider range of scalar types and for matrix or quaternion
operations.
