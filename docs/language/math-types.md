# First-class SIMD Mathematical Types

Singulisp provides vector, quaternion, and matrix types for high-performance numerical computation as **first-class built-in types**. Like `exp` and `sqrt`, they are built-ins retained as first-class operations in IR without trait-based dynamic dispatch. The backend lowers them to SIMD instructions, LLVM intrinsics, libm, or corresponding fast kernels.

Every mathematical type has **Copy semantics** and is copied automatically on assignment and when passed to a function. They may also be stored in arrays and struct fields; the backend and use context determine their actual placement on the stack, in registers, or in memory.

---

## Two Coexisting Families

Singulisp has two families of math-related types:

| Family | Example types | Uses |
|------|-------|------|
| **High-level mathematical types** (the subject of this chapter) | `Vec3`, `Quat`, `Mat4`, and others | Numerical computation for geometry, transformations, physics, graphics, and similar domains |
| **Low-level SIMD vector types** | `f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, `u32x4` | Lane-parallel computation in SPMD and kernel code |

High-level mathematical types preserve type-specific meaning, while low-level SIMD types express explicit lane operations.

---

## Type List and Physical Representation

### Vector Types

| Type | Components | LLVM type | Bytes | Notes |
|------|------|---------|---------|------|
| `Vec2` | x, y | `<2 x float>` | 8 | 2D coordinates and UV |
| `Vec3` | x, y, z | `<4 x float>` | 16 | 3D coordinates and normals (w lane unused) |
| `Vec4` | x, y, z, w | `<4 x float>` | 16 | Homogeneous coordinates and RGBA |
| `Quat` | x, y, z, w | `<4 x float>` | 16 | Unit quaternion (assumes x²+y²+z²+w²=1) |
| `Float3` | x, y, z | `{f32, f32, f32}` | 12 | Packed struct, **non-SIMD**, exclusively for array storage |

Every component has type **f32**. There are no f64 vector types.

### Matrix Types

| Type | Storage order | LLVM type | Bytes | Notes |
|------|-------|---------|---------|------|
| `Mat3` | Column-major | `[3 x <4 x float>]` | 48 | OpenGL convention |
| `Mat3r` | Row-major | `[3 x <4 x float>]` | 48 | Direct3D convention |
| `Mat4` | Column-major | `[4 x <4 x float>]` | 64 | General-purpose transformation matrix |
| `Mat4r` | Row-major | `[4 x <4 x float>]` | 64 | For HLSL/Direct3D |
| `Mat3x4` | Row-major affine (3 rows × 4 columns) | `[3 x <4 x float>]` | 48 | Omits the final 4×4 row `(0,0,0,1)` |

Column-major and row-major types have the same LLVM memory layout: a sequence of `<4 x float>`. Only their interpretation and operation formulas differ.

### Reserved Type Names

`Vec2`, `Vec3`, `Vec4`, `Quat`, `Float3`, `Mat3`, `Mat3r`, `Mat4`, `Mat4r`, and `Mat3x4` are all reserved type names. Defining a user `defstruct` or `deftype` with any of these names produces compile-time error `E-IR-0150`.

---

## Construction

### Vectors, Quaternions, and `Float3`

The constructor is the type name itself. A component-count mismatch produces `E-MATH-0001`.

```singulisp
(let a (Vec2 1.0f 2.0f))
(let b (Vec3 1.0f 2.0f 3.0f))
(let c (Vec4 1.0f 2.0f 3.0f 4.0f))
(let q (Quat 0.0f 0.0f 0.0f 1.0f))   ;; x y z w (identity quaternion)
(let f (Float3 1.5f 2.5f 3.5f))
```

### Matrices

The constructor is the type name itself. Pass column vectors to column-major matrices (`Mat3`/`Mat4`), and row vectors to row-major matrices (`Mat3r`/`Mat4r`) and `Mat3x4`. A column- or row-vector count mismatch produces `E-MATH-0004`.

```singulisp
;; Mat3 (column-major): three column vectors (Vec3)
(let m3 (Mat3 (Vec3 1.0f 0.0f 0.0f)
              (Vec3 0.0f 1.0f 0.0f)
              (Vec3 0.0f 0.0f 1.0f)))

;; Mat4 (column-major): four column vectors (Vec4)
(let m4 (Mat4 (Vec4 1.0f 0.0f 0.0f 0.0f)
              (Vec4 0.0f 1.0f 0.0f 0.0f)
              (Vec4 0.0f 0.0f 1.0f 0.0f)
              (Vec4 0.0f 0.0f 0.0f 1.0f)))

;; Mat4r (row-major): four row vectors (Vec4)
(let mr (Mat4r (Vec4 1.0f 0.0f 0.0f 0.0f)
               (Vec4 0.0f 1.0f 0.0f 0.0f)
               (Vec4 0.0f 0.0f 1.0f 0.0f)
               (Vec4 0.0f 0.0f 0.0f 1.0f)))

;; Mat3x4 (row-major affine): three row vectors (Vec4)
;; The fourth component (w) of each Vec4 is a translation component
(let aff (Mat3x4 (Vec4 1.0f 0.0f 0.0f 10.0f)
                 (Vec4 0.0f 1.0f 0.0f 20.0f)
                 (Vec4 0.0f 0.0f 1.0f 30.0f)))
```

---

## Component Access

Use dot notation to extract a lane. The result has type **f32**.

| Component | Lane | Vec2 | Vec3 | Vec4 | Quat | Float3 |
|------|-------|------|------|------|------|--------|
| `.x` | 0 | ○ | ○ | ○ | ○ | ○ |
| `.y` | 1 | ○ | ○ | ○ | ○ | ○ |
| `.z` | 2 | × | ○ | ○ | ○ | ○ |
| `.w` | 3 | × | × | ○ | ○ | × |

An out-of-range component, such as `.z` on `Vec2`, produces compile-time error `E-MATH-0010`.

```singulisp
(let v (Vec3 1.0f 2.0f 3.0f))
v.x   ;; → f32 = 1.0f
v.y   ;; → f32 = 2.0f
v.z   ;; → f32 = 3.0f

(let q (Quat 0.0f 0.0f 0.0f 1.0f))
q.w   ;; → f32 = 1.0f (scalar part)
```

Matrix types have no component access such as `.x`. Use `MatN/col-N` to retrieve a column.

---

## Vector Operations

### Calling Convention

Use the namespaced form `(TypeName/operation-name argument...)`.

```singulisp
(Vec3/add a b)
(Vec3/dot a b)
```

Below, `VecN` denotes `Vec2`, `Vec3`, or `Vec4`; `cross` is available only for `Vec3`. `Quat` also supports some general vector operations, but multiplication and angle have quaternion-specific semantics and are covered separately below.

### Arithmetic Operations (Component-wise for Vec2 / Vec3 / Vec4)

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `VecN/add` | 2 | VecN | Component-wise addition |
| `VecN/sub` | 2 | VecN | Component-wise subtraction |
| `VecN/mul` | 2 | VecN | Component-wise multiplication (Hadamard product) |
| `VecN/div` | 2 | VecN | Component-wise division |
| `VecN/min` | 2 | VecN | Component-wise minimum |
| `VecN/max` | 2 | VecN | Component-wise maximum |
| `VecN/scale` | 2 (v, f32) | VecN | Scalar multiplication |
| `VecN/neg` | 1 | VecN | Negation |
| `VecN/abs` | 1 | VecN | Component-wise absolute value |
| `VecN/splat` | 1 (f32) | VecN | Constructs with the same value in every component |

```singulisp
(Vec3/scale v 2.0f)    ;; scalar multiplication
(Vec3/splat 1.0f)      ;; (Vec3 1.0f 1.0f 1.0f)
```

### Geometric Operations

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `VecN/dot` | 2 | f32 | Dot product |
| `Vec3/cross` | 2 | Vec3 | Cross product (Vec3 only; `E-MATH-0003` for other types) |
| `VecN/length-sq` | 1 | f32 | Squared length (no square root; fast) |
| `VecN/length` | 1 | f32 | Length |
| `VecN/normalize` | 1 | VecN | Normalization to length 1 |
| `VecN/distance` | 2 | f32 | Distance between two points |
| `VecN/distance-sq` | 2 | f32 | Squared distance between two points |
| `VecN/angle` | 2 | f32 | Angle between two vectors in radians |
| `VecN/reflect` | 2 (i, n) | VecN | Reflection: `i - 2*dot(i,n)*n` |
| `VecN/refract` | 3 (i, n, eta) | VecN | Refraction (Snell's law); zero vector under total internal reflection |
| `VecN/faceforward` | 3 (n, i, nref) | VecN | `dot(nref,i)<0 ? n : -n` |
| `VecN/project` | 2 (a, b) | VecN | Projection onto b: `b*dot(a,b)/dot(b,b)` |
| `VecN/reject` | 2 (a, b) | VecN | Orthogonal component: `a - project(a,b)` |

### Interpolation and Clamping

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `VecN/lerp` | 3 (a, b, t:f32) | VecN | Linear interpolation: `a + (b-a)*t` |
| `VecN/clamp` | 3 (v, lo, hi) | VecN | Component-wise clamping |
| `VecN/smoothstep` | 3 (e0, e1, x) | VecN | Component-wise Hermite interpolation |
| `VecN/smootherstep` | 3 (e0, e1, x) | VecN | Smoother interpolation (Ken Perlin's 6t⁵-15t⁴+10t³) |
| `VecN/inverse-lerp` | 3 (a, b, v) | VecN | Inverse linear interpolation: `(v-a)/(b-a)` |
| `VecN/remap` | 5 (x, in0, in1, out0, out1) | VecN | Range remapping |

### Color-space Conversion (Vec3-specific Semantics)

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `Vec3/linear-to-srgb` | 1 | Vec3 | Linear → sRGB (gamma correction) |
| `Vec3/srgb-to-linear` | 1 | Vec3 | sRGB → linear (gamma removal) |
| `Vec3/to-hsv` | 1 | Vec3 | RGB → HSV (each component 0..1) |
| `Vec3/to-rgb` | 1 | Vec3 | HSV → RGB |

### Predicates

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `VecN/all` | 1 | bool | Whether every component is nonzero |
| `VecN/any` | 1 | bool | Whether any component is nonzero |

### Component-wise Mathematical Functions (Unary)

Calling one of the following functions in the form `(VecN/op v)` returns a vector of the same type with the function applied to each component.

| Operation | Meaning |
|-------|------|
| `sin`, `cos`, `tan` | Trigonometric functions |
| `asin`, `acos`, `atan` | Inverse trigonometric functions |
| `sinh`, `cosh`, `tanh` | Hyperbolic functions |
| `asinh`, `acosh`, `atanh` | Inverse hyperbolic functions |
| `exp`, `exp2`, `exp10` | Exponential functions |
| `expm1` | `e^x - 1` |
| `ln`, `log2`, `log10` | Logarithmic functions |
| `log1p` | `ln(1+x)` |
| `sqrt`, `cbrt` | Square and cube roots |
| `floor`, `ceil`, `round`, `trunc` | Rounding |
| `sign` | Sign (-1/0/1) |
| `fract` | Fractional part: `x - floor(x)` |
| `saturate` | `clamp(x, 0, 1)` |
| `degrees`, `radians` | Unit conversion |
| `abs` | Absolute value (identical to `VecN/abs` in the arithmetic section) |

### Component-wise Mathematical Functions (Binary)

| Operation | Arguments | Meaning |
|-------|--------|------|
| `VecN/pow` | 2 (base, exp) | Component-wise exponentiation |

---

## Quaternion Operations

Quaternion-specific operations are provided in the `Quat/` namespace. `Quat` also supports some general vector operations, such as `Quat/add`, `Quat/dot`, and `Quat/normalize`, but its specialized dispatcher resolves first. In particular, `Quat/mul` is a Hamilton product rather than component-wise multiplication, and `Quat/angle` is the angular difference between two rotations.

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `Quat/identity` | 0 | Quat | Identity quaternion `(0,0,0,1)` |
| `Quat/mul` | 2 (a, b) | Quat | Hamilton product (rotation composition) |
| `Quat/conjugate` | 1 | Quat | Conjugate quaternion (inverse rotation for a unit Quat) |
| `Quat/inverse` | 1 | Quat | Inverse quaternion: `conjugate(q)/dot(q,q)` |
| `Quat/from-axis-angle` | 2 (axis:Vec3, angle:f32) | Quat | Constructs from an axis and angle |
| `Quat/from-euler` | 3 (x, y, z: f32 radians) | Quat | Constructs from Euler angles in ZYX order |
| `Quat/from-to-rotation` | 2 (from:Vec3, to:Vec3) | Quat | Shortest rotation from `from` to `to` |
| `Quat/look-rotation` | 2 (forward:Vec3, up:Vec3) | Quat | Creates a rotation from forward and up directions |
| `Quat/rotate-vec3` | 2 (q, v:Vec3) | Vec3 | Rotates a vector by a quaternion |
| `Quat/to-mat4` | 1 | Mat4 | Converts to a Mat4 rotation matrix |
| `Quat/to-euler` | 1 | Vec3 | Converts to Euler angles (x/y/z radians) |
| `Quat/slerp` | 3 (a, b, t:f32) | Quat | Spherical linear interpolation |
| `Quat/nlerp` | 3 (a, b, t:f32) | Quat | Normalized linear interpolation (shortest path; fast) |
| `Quat/angle` | 2 (a, b) | f32 | Angular difference between two quaternions in radians |
| `Quat/delta` | 2 (a, b) | Quat | Delta rotation: `b * inverse(a)` |
| `Quat/twist` | 2 (q, axis:Vec3) | Quat | Twist component of a swing–twist decomposition |
| `Quat/scale-rotation` | 2 (q, t:f32) | Quat | Scales a rotation by t (`q^t`) |
| `Quat/with-angle` | 2 (q, angle:f32) | Quat | Sets the angle while preserving the rotation axis |

```singulisp
;; 90° rotation around the Y axis
(let q (Quat/from-axis-angle (Vec3 0.0f 1.0f 0.0f)
                              (* 3.14159265f 0.5f)))

;; Rotate the X-axis direction
(let r (Quat/rotate-vec3 q (Vec3 1.0f 0.0f 0.0f)))

;; Half rotation (interpolated with slerp)
(let q-half (Quat/slerp (Quat/identity) q 0.5f))

;; Convert to a 4x4 matrix
(let m (Quat/to-mat4 q))
```

---

## Matrix Operations

Matrix operations are provided in the `Mat3/`, `Mat3r/`, `Mat4/`, `Mat4r/`, and `Mat3x4/` namespaces.

### Basic Operations (Square Matrix Types)

In this table, `MatN` denotes `Mat3`, `Mat3r`, `Mat4`, or `Mat4r`. `Mat3x4` is 3×4 row-major affine storage and is not included in the common square-matrix operations.

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `MatN/identity` | 0 | MatN | Identity matrix |
| `MatN/mul` | 2 (a, b) | MatN | Matrix product |
| `MatN/transpose` | 1 | MatN | Transpose |
| `MatN/transform` | 2 (m, v) | VecK | Matrix × vector transformation (\*1) |
| `MatN/orthonormalize` | 1 | MatN | Gram–Schmidt orthonormalization |

(\*1) Vector types: Mat3/Mat3r→Vec3, Mat4/Mat4r→Vec4

### Construction Helpers

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `Mat4/Mat4r/Mat3x4/from-translation` | 1 (Vec3) | Same type | Translation matrix (unavailable for 3x3: `E-MATH-0008`) |
| `MatN/from-scale` | 1 (Vec3) | MatN | Scaling matrix |
| `Mat3/Mat3r/Mat4/Mat4r/Mat3x4/from-quat` | 1 (Quat) | Same type | Rotation matrix from a quaternion |
| `Mat4/Mat4r/Mat3x4/from-trs` | 3 (Vec3, Quat, Vec3) | Same type | Composition from translation, rotation, and scale |
| `Mat4/perspective` | 4 (fovy, aspect, near, far) | Mat4 | Perspective projection matrix |
| `Mat4/ortho` | 6 (l, r, b, t, near, far) | Mat4 | Orthographic projection matrix |
| `Mat4/frustum` | 6 (l, r, b, t, near, far) | Mat4 | General frustum projection matrix |
| `Mat4/look-at` | 3 (eye, center, up: Vec3) | Mat4 | Right-handed view matrix |

### Inverse, Determinant, and Decomposition

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `Mat4/inverse` | 1 | Mat4 | General 4×4 inverse (cofactor expansion) |
| `Mat3/inverse` | 1 | Mat3 | 3×3 inverse |
| `Mat4/determinant` | 1 | f32 | Determinant |
| `Mat3/determinant` | 1 | f32 | 3×3 determinant |
| `Mat3/from-mat4` | 1 (Mat4) | Mat3 | Extracts the upper-left 3×3 of a Mat4 for a normal matrix |
| `Mat4/get-translation` | 1 | Vec3 | Extracts the translation component |
| `Mat4/get-scale` | 1 | Vec3 | Extracts scale components as the length of each basis column |
| `Mat4/get-rotation` | 1 | Quat | Extracts the rotation component as a quaternion |

### Mat3x4-specific (Affine Matrix)

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `Mat3x4/identity` | 0 | Mat3x4 | Affine identity matrix |
| `Mat3x4/from-translation` | 1 (Vec3) | Mat3x4 | Stores translation in the fourth component of each row |
| `Mat3x4/from-scale` | 1 (Vec3) | Mat3x4 | Affine scaling matrix |
| `Mat3x4/from-trs` | 3 (Vec3, Quat, Vec3) | Mat3x4 | Row-major TRS affine matrix |
| `Mat3x4/from-quat` | 1 (Quat) | Mat3x4 | Constructs an affine rotation matrix from a quaternion |
| `Mat3x4/mul` | 2 (Mat3x4, Mat3x4) | Mat3x4 | Composes affine matrices |
| `Mat3x4/transform` | 2 (aff, point:Vec3) | Vec3 | Affine transform (dot product of each row with (p,1)) |
| `Mat3x4/orthonormalize` | 1 | Mat3x4 | Orthonormalizes the affine basis |
| `Mat3x4/inverse` | 1 | Mat3x4 | Affine inverse transform |
| `Mat3x4/get-translation` | 1 | Vec3 | Retrieves the fourth component of each row |
| `Mat3x4/get-scale` | 1 | Vec3 | Retrieves the scale components |
| `Mat3x4/get-rotation` | 1 | Quat | Retrieves the rotation component |

Each `Mat3x4` operation treats the value as an affine matrix with an implied bottom row `(0, 0, 0, 1)`. No same-type `transpose` or unqualified `determinant` is defined.

### Direct Column Retrieval (for Serialization)

| Operation | Arguments | Result | Meaning |
|-------|--------|--------|------|
| `Mat3`, `Mat3r`, `Mat3x4`: `col-0` through `col-2` | 1 | Vec4 | Retrieves one of three storage vectors |
| `Mat4`, `Mat4r`: `col-0` through `col-3` | 1 | Vec4 | Retrieves one of four storage vectors |

```singulisp
;; Compose translation and scale
(let m (Mat4/mul (Mat4/from-translation (Vec3 10.0f 20.0f 30.0f))
                 (Mat4/from-scale (Vec3 2.0f 3.0f 4.0f))))

;; Transform a point (w=1)
(let p (Mat4/transform m (Vec4 1.0f 1.0f 1.0f 1.0f)))

;; Restore the original point with the inverse matrix
(let back (Mat4/transform (Mat4/inverse m) p))

;; Perspective projection matrix
(let proj (Mat4/perspective 1.0472f 1.7778f 0.1f 1000.0f))

;; View matrix
(let view (Mat4/look-at (Vec3 0.0f 5.0f 10.0f)
                        (Vec3 0.0f 0.0f 0.0f)
                        (Vec3 0.0f 1.0f 0.0f)))
```

---

## Float3 ⇔ Vec3 Conversion

`Float3` is a packed type exclusively for array storage (12B, 4B aligned). Because the LLVM representation of `Vec3` is `<4 x float>` (16B), it includes padding. `Float3` allows dense storage in `Vec<Float3>`, improving cache efficiency.

**SIMD operations cannot be performed on it directly. Convert it to `Vec3` for computation.**

| Operation | Argument | Result | Meaning |
|-------|------|--------|------|
| `Vec3/of-float3` | Float3 | Vec3 | Float3 → Vec3 (fills w with 0) |
| `Float3/of-vec3` | Vec3 | Float3 | Vec3 → Float3 (discards w) |

```singulisp
;; Store Float3 in Vec (densely packed at 12B)
(let mut positions : [Vec Float3] (Vec/new))
(Vec/push (& mut positions) (Float3 1.0f 2.0f 3.0f))

;; Convert to Vec3 for computation
(let p (Vec/get (& positions) 0))
(let v3 (Vec3/scale (Vec3/of-float3 p) 2.0f))

;; Write back
(Vec/set (& mut positions) 0 (Float3/of-vec3 v3))
```

`Float3` also supports dot-notation access (`.x`, `.y`, `.z`).

---

## Packing Operations (GPU Transfer and Vertex Compression)

| Operation | Argument | Result | Meaning |
|-------|------|--------|------|
| `pack/half2x16` | Vec2 | u32 | Packs Vec2 as 2×f16 into u32 |
| `unpack/half2x16` | u32 | Vec2 | Unpacks two f16 values from a u32 into a Vec2 |
| `pack/unorm4x8` | Vec4 (0..1) | u32 | Packs Vec4 as 4×unorm8 into u32 |
| `unpack/unorm4x8` | u32 | Vec4 | Unpacks four unorm8 values from a u32 into a Vec4 (0..1) |
| `pack/snorm4x8` | Vec4 (-1..1) | u32 | Packs Vec4 as 4×snorm8 into u32 |
| `unpack/snorm4x8` | u32 | Vec4 | Unpacks four snorm8 values from a u32 into a Vec4 (-1..1) |

```singulisp
;; f16 half-precision packing (GPU vertex format)
(let packed (pack/half2x16 (Vec2 1.0f 0.5f)))   ;; → u32
(let v (unpack/half2x16 packed))                  ;; → Vec2

;; RGBA unorm8 packing (texture write)
(let p4 (pack/unorm4x8 (Vec4 1.0f 0.5f 0.0f 1.0f)))   ;; → u32
(let v4 (unpack/unorm4x8 p4))                            ;; → Vec4
```

---

## Scalar Mathematical Functions (f32 / f64)

Scalar mathematical functions are provided in the `f32/` and `f64/` namespaces. They are retained as first-class mathematical operations in IR, and the backend lowers them to LLVM intrinsics, libm, or supported fast kernels available under `#[fp-mode :fast]`. Not every mode and operation is independent of libm.

### Unary Functions

| Function | Meaning |
|-------|------|
| `f32/sqrt` | Square root |
| `f32/cbrt` | Cube root |
| `f32/abs` | Absolute value |
| `f32/sign` | Sign (-1.0/0.0/+1.0) |
| `f32/fract` | Fractional part: `x - floor(x)` |
| `f32/saturate` | `clamp(x, 0, 1)` |
| `f32/sin`, `f32/cos`, `f32/tan` | Trigonometric functions |
| `f32/asin`, `f32/acos`, `f32/atan` | Inverse trigonometric functions (one argument) |
| `f32/sinh`, `f32/cosh`, `f32/tanh` | Hyperbolic functions |
| `f32/asinh`, `f32/acosh`, `f32/atanh` | Inverse hyperbolic functions |
| `f32/exp`, `f32/exp2`, `f32/exp10` | Exponential functions |
| `f32/expm1` | `e^x - 1` (improved precision near x ≈ 0) |
| `f32/ln`, `f32/log2`, `f32/log10` | Logarithmic functions |
| `f32/log1p` | `ln(1+x)` (improved precision near x ≈ 0) |
| `f32/floor`, `f32/ceil`, `f32/round`, `f32/trunc` | Rounding |
| `f32/degrees` | Radians → degrees |
| `f32/radians` | Degrees → radians |
| `f32/isnan` | NaN test → bool |
| `f32/isinf` | Infinity test → bool |
| `f32/isfinite` | Finite-value test → bool |

The f64 versions are likewise provided as `f64/sqrt` and so on.

### Binary Functions

| Function | Meaning |
|-------|------|
| `f32/min`, `f32/max` | Minimum and maximum |
| `f32/pow` | Exponentiation `x^y` |
| `f32/atan2` | Two-argument arctangent (all quadrants, range (-π, π]) |
| `f32/copysign` | Magnitude of x with the sign of y |
| `f32/hypot` | `sqrt(x²+y²)` with overflow resistance |
| `f32/step` | `x >= edge ? 1.0 : 0.0` |
| `f32/fmod` | Floating-point remainder: `x - trunc(x/y)*y` |
| `f32/repeat` | Wraps to `[0, length)` |
| `f32/wrap-signed` | Wraps to `[-length/2, length/2)` |
| `f32/ping-pong` | Oscillates between `0↔length` |
| `f32/delta-angle` | Shortest signed difference between two angles in radians, range `[-π, π)` |

### Ternary and Multi-argument Functions

| Function | Arguments | Meaning |
|-------|--------|------|
| `f32/fma` | 3 (a, b, c) | Fused multiply-add `a*b+c` (LLVM `llvm.fma`) |
| `f32/lerp` | 3 (a, b, t) | Linear interpolation: `a + (b-a)*t` |
| `f32/clamp` | 3 (x, lo, hi) | Clamping |
| `f32/smoothstep` | 3 (e0, e1, x) | Hermite interpolation (3t²-2t³) |
| `f32/smootherstep` | 3 (e0, e1, x) | Ken Perlin's fifth-order interpolation (6t⁵-15t⁴+10t³) |
| `f32/inverse-lerp` | 3 (a, b, v) | Inverse linear interpolation: `(v-a)/(b-a)` |
| `f32/lerp-angle` | 3 (from, to, t) | Shortest-path angular interpolation |
| `f32/remap` | 5 (x, in0, in1, out0, out1) | Remaps interval [in0,in1] to [out0,out1] |

### Bit Operations

| Function | Meaning |
|-------|------|
| `f32/to-bits` | f32 → u32 (bit reinterpretation) |
| `f32/from-bits` | u32 → f32 |
| `f64/to-bits` | f64 → u64 |
| `f64/from-bits` | u64 → f64 |

### Constants (Zero Arguments)

| Constant | Value |
|-------|-----|
| `f32/PI` | π ≈ 3.14159265 |
| `f32/TAU` | 2π ≈ 6.28318530 |
| `f32/E` | e ≈ 2.71828182 |
| `f32/INF` | +∞ |
| `f32/NAN` | NaN |
| `f32/MAX` | Largest positive finite value ≈ 3.40282347e38 |
| `f32/MIN` | Most negative finite value ≈ -3.40282347e38 |
| `f32/EPSILON` | Machine epsilon for f32 ≈ 1.19209290e-7 |

The f64 versions are likewise provided as `f64/PI` and so on. Passing an argument produces error `E-IR-0050`.

---

## Integer Bit Operations

The following bit operations are available on integer types such as `i32/`, `u32/`, `i64/`, and `u64/`.

### Unary

| Function | Meaning |
|-------|------|
| `i32/popcount` | Number of set bits (popcount instruction) |
| `i32/clz` | Number of leading zero bits |
| `i32/ctz` | Number of trailing zero bits |
| `i32/bswap` | Reverses byte order (endian conversion) |
| `u32/reverse-bits` | Reverses the order of all bits |

### Binary

| Function | Meaning |
|-------|------|
| `u32/rotl` | Rotate left |
| `u32/rotr` | Rotate right |
| `i32/sat-add` | Saturating addition (clamped at maximum) |
| `u32/sat-add` | Unsigned saturating addition (clamped at maximum) |
| `i32/sat-sub` | Saturating subtraction (clamped at minimum) |
| `u32/sat-sub` | Unsigned saturating subtraction (clamped at 0) |

`sat-add` and `sat-sub` are provided for every integer width: `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, and `u64`.

---

## Type Annotations and Use in Structs

Like ordinary types, mathematical types may be used in type annotations, struct fields, and as `Vec` element types.

```singulisp
;; Type annotation
(defn normalize-v [(v : Vec3)] -> Vec3
  (Vec3/normalize v))

;; Struct fields
(defstruct Entity
  (pos    : Vec3)
  (vel    : Vec3)
  (orient : Quat))

;; Vec elements
(let mut ps : [Vec Vec3] (Vec/new))
(Vec/push (& mut ps) (Vec3 1.0f 2.0f 3.0f))

;; Nested field access
(let e (Entity (Vec3 0.0f 0.0f 0.0f)
               (Vec3 1.0f 0.0f 0.0f)
               (Quat/identity)))
e.pos.x   ;; → f32

;; Use in const
(const GRAVITY  : Vec3 (Vec3 0.0f -9.81f 0.0f))
(const IDENTITY : Mat4 (Mat4/identity))
```

---

## Static Dispatch and Fast Math

Every mathematical-type operation uses **static dispatch**. It is retained as a dedicated `MathOp`, `MathUnary`, or `MathBinary` IR node, and the backend selects vector instructions, intrinsics, libm calls, or a specialized fast kernel according to the target and operation. No dynamic dispatch through traits occurs.

Attaching the `#[fp-mode :fast]` attribute to a function enables LLVM fast-math flags and promotes vectorization:

```singulisp
#[fp-mode :fast]
(defn step [(p : Vec3) (v : Vec3) (dt : f32)] -> Vec3
  (Vec3/add p (Vec3/scale v dt)))
```

---

## No Import Required

Mathematical types such as `Vec2`, `Vec3`, `Vec4`, `Quat`, `Mat3`, and `Mat4` are **always in scope** and require no import through `use`.

---

## Error Codes

| Code | Meaning |
|-------|------|
| `E-MATH-0001` | Invalid component count for a Vec/Quat/Float3 constructor |
| `E-MATH-0002` | Invalid vector-operation argument count, or first argument is not a vector type |
| `E-MATH-0003` | `cross` applied to a type other than `Vec3`; it is defined only for `Vec3` |
| `E-MATH-0004` | Invalid number of column or row vectors in a matrix constructor |
| `E-MATH-0007` | Invalid argument count for a Quat-specific operation |
| `E-MATH-0008` | Translation operation (`from-translation`/`from-trs`) applied to a 3×3 matrix (Mat3/Mat3r) |
| `E-MATH-0009` | Same-type transpose or determinant applied to `Mat3x4` |
| `E-MATH-0010` | Access to a nonexistent component name (`.x`/`.y`/`.z`/`.w`) |
| `E-IR-0050` | Argument passed to a constant such as INF or NAN |
| `E-IR-0150` | User-defined type declared with a reserved type name |
