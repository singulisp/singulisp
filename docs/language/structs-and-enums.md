# Structs and Enums

Singulisp provides **structs** (`defstruct`) and **algebraic data types / enums** (`deftype`) as user-defined types. It also provides mechanisms such as SIMD alignment, GPU-compatible layouts, and SoA collections for defining efficient data representations.

---

## `defstruct` (Struct Definitions)

### Syntax

```lisp
(defstruct StructName
  (field-name : type)
  ...)
```

### Basic Examples

```lisp
(defstruct Point
  (x : f32)
  (y : f32))
```

```lisp
(defstruct Player
  (name     : String)
  (hp       : i32)
  (position : Point))
```

---

## Struct Literals

Create a struct value with constructor syntax. Fields appear in definition order. There are no named arguments of the form `:key value`.

```lisp
(let origin (Point 0.0f 0.0f))
(let player (Player "Hero" 100 (Point 5.0f 3.0f)))
```

---

## Field Access

Access a field with dot (`.`) notation.

```lisp
(let p (Point 1.0f 2.0f))
(println! "x = {}" p.x)     ; => x = 1.0
(println! "y = {}" p.y)     ; => y = 2.0
```

Dot notation may be chained to access nested fields.

```lisp
(println! "player x = {}" player.position.x)
```

---

## Visibility Attributes

A struct may have a visibility annotation. Visibility applies to the entire struct and cannot be specified separately for individual fields.

| Attribute | Visibility |
|------|---------|
| `#[pub]` | Accessible from outside the package |
| (default) | Within the same package only |
| `#[priv]` | Within the same folder only |

```lisp
#[pub]
(defstruct Color
  (r : u32)
  (g : u32)
  (b : u32)
  (a : u32))
```

---

## `repr` Specifications

A `repr` specification on `defstruct` controls memory layout for performance, ABI, and GPU compatibility.

### :repr simd

This is a hint requiring 16-byte alignment at the corresponding stack or arena allocation site. It does not change the type's ABI alignment or tail padding, so the size and array stride of a struct containing three `f32` fields remain 12 bytes.

```lisp
#[pub]
(defstruct Position3 :repr simd
  (x : f32)
  (y : f32)
  (z : f32))

#[pub]
(defstruct Color4 :repr simd
  (x : f32)
  (y : f32)
  (z : f32)
  (w : f32))
```

### :repr gpu

Follows std430 layout rules to ensure compatibility with GPU buffers. The GPU checker and code generator recognize `:repr gpu` directly.

```lisp
(defstruct Particle :repr gpu
  (position : Vec3)
  (velocity : Vec3)
  (life     : f32))
```

### :repr C

A C-compatible memory layout, used when sharing a struct with C through FFI (Foreign Function Interface).

```lisp
(defstruct CHeader :repr C
  (magic   : u32)
  (version : u32)
  (flags   : u32))
```

### :repr packed

Lays out fields in declaration order without padding. Reads and writes of a field that does not meet its natural alignment use unaligned access. `Float3` is a distinct built-in type and is not a substitute for `:repr packed`.

```lisp
(defstruct PackedData :repr packed
  (tag  : u32)
  (len  : u32)
  (data : u32))
```

### :repr ordered

Suppresses compiler field-reordering optimizations and preserves declaration-order layout.

### :repr aligned N

Pads the type's size to an N-byte boundary. N must be a power of two. This is used for cache-line efficiency and arena-allocation alignment guarantees. It may be used with both `defstruct` and `deftype`.

There is no `#[layout]` attribute (E-SOA-0053). All layout specifications use inline `:repr` modifiers.

```lisp
(defstruct Node :repr aligned 32
  (left  : [& Tree])
  (right : [& Tree])
  (value : i32))   ; → padded to 32B even when the natural size is 20B

(deftype Tree :repr aligned 16
  Leaf
  (Node [& Tree] [& Tree]))  ; → padded to a 16B boundary
```

Alignment is guaranteed for both stack variables and arena allocations. Array elements automatically inherit the alignment as well.

---

## `deftype` (Algebraic Data Types / Enums)

### Syntax

```lisp
(deftype TypeName
  (VariantName field-type...)
  (VariantName field-type...)
  (VariantName))

; Definitions with named fields are also supported
(deftype TypeName
  (VariantName (field-name : field-type) ...)
  ...)

; With a repr specification
(deftype TypeName :repr aligned N
  (VariantName field-type...)
  ...)
```

Each variant may have zero or more fields. Named fields on `deftype` improve readability at the definition site; value construction and `match` bindings remain positional.

The only representation allowed on `deftype` is `:repr aligned N`. `C`, `simd`, `gpu`, `packed`, and `ordered` are exclusive to `defstruct`; specifying one on `deftype` is a compile-time error.

### Basic Example

```lisp
(deftype Shape
  (Circle f32)
  (Rect f32 f32)
  (Point))
```

- `Circle` has one `f32` value (the radius).
- `Rect` has two `f32` values (the width and height).
- `Point` has no fields.

### Constructing Values

```lisp
(let s1 (Circle 5.0f))
(let s2 (Rect 3.0f 4.0f))
(let s3 Point)           ; fieldless variant
```

### Pattern Matching

Branch on an enum value with `match`.

```lisp
(defn area [(shape : Shape)] -> f32
  (match shape
    [(Circle r) (* 3.14159f (* r r))]
    [(Rect w h) (* w h)]
    [Point 0.0f]))
```

### Recursive Data Types

Use `Box` to define a recursive enum.

```lisp
(deftype Expr
  (Lit i32)
  (Add [Box Expr] [Box Expr])
  (Mul [Box Expr] [Box Expr]))

(defn eval-expr [(e : Expr)] -> i32
  (match e
    [(Lit n)     n]
    [(Add l r)   (+ (eval-expr (* l)) (eval-expr (* r)))]
    [(Mul l r)   (* (eval-expr (* l)) (eval-expr (* r)))]))
```

### Generic Enums

An enum may also have type parameters. The built-in `Option` and `Result` are conceptually defined as follows.

```lisp
; Conceptual definitions (already provided as built-in types)
(deftype Option [T]
  (Some T)
  (None))

(deftype Result [T E]
  (Ok T)
  (Err E))
```

### Built-in ADTs

The compiler predefines the following ADTs; users cannot redefine them.

| Type | Variants |
|----|-----------|
| `[Option T]` | `None` / `(Some T)` |
| `[Result T E]` | `(Ok T)` / `(Err E)` |
| `AllocError` | `AllocOom` |
| `DeserError` | `(Truncated i32)` / `(BadTag i32)` / `BadSchema` / `CycleDetected` / `(TooLarge i32)` |

`AllocError` is returned when a region encounters OOM. `DeserError` is the detailed error type returned by binary deserialization (`T/try-deserialize`).

---

## #[derive]

The `#[derive]` attribute automatically derives implementation functions for well-known traits on structs and enums.

```lisp
#[derive Eq]
(defstruct Tile
  (x : i32)
  (y : i32)
  (kind : u32))
```

### Derivable Traits

| Trait | Effect |
|---------|------|
| `Eq` | Equality comparison |
| `Hash` | Deterministic `u64` hash |
| `Ord` | Lexicographic comparison based on declaration order |

Structs use field declaration order, and enums use variant declaration order, as the basis for comparison and hash generation.

---

## SoA collection

Singulisp represents SoA (Structure of Arrays) as a **collection type**, not a struct attribute.

- The `Particle` struct itself is an ordinary value type.
- A fixed-length SoA is `[SoaArray Particle N]`.
- A variable-length SoA is `[SoaVec Particle]`.
- `get` and `set`, which would implicitly read or write a whole struct in an SoA collection, are not available.
- Normally, use `get-field` and `set-field`; use explicit `gather` and `scatter` only where a whole struct is required.

### AoS vs SoA

An ordinary AoS array:

```
[pos0, rot0, scl0] [pos1, rot1, scl1] [pos2, rot2, scl2] ...
```

SoA collection:

```
[pos0, pos1, pos2, ...] [rot0, rot1, rot2, ...] [scl0, scl1, scl2, ...]
```

This improves cache efficiency and SIMD load efficiency for operations that traverse only particular fields.

### Example

```lisp
(defstruct Transform
  (position : Vec3)
  (rotation : Quat)
  (scale    : Vec3))

(defn read-scale [] -> Vec3
  (let xs : [SoaArray Transform 2]
    (array
      (Transform (Vec3 0.0f 0.0f 0.0f) (Quat 0.0f 0.0f 0.0f 1.0f) (Vec3 1.0f 1.0f 1.0f))
      (Transform (Vec3 1.0f 0.0f 0.0f) (Quat 0.0f 0.0f 0.0f 1.0f) (Vec3 2.0f 2.0f 2.0f))))
  (SoaArray/get-field xs 1 scale))
```

SoA is represented as a collection type, not as a struct attribute.

---

## `impl` Blocks

Use `impl` to add methods to a struct or enum.

```lisp
(defstruct Circle
  (center : Point)
  (radius : f32))

(impl Circle
  (defn new [(center : Point) (radius : f32)] -> Circle
    (Circle center radius))

  (defn area [(self : Circle)] -> f32
    (* 3.14159f (* self.radius self.radius)))

  (defn contains? [(self : Circle) (p : Point)] -> bool
    (let dx (- p.x self.center.x))
    (let dy (- p.y self.center.y))
    (<= (+ (* dx dx) (* dy dy)) (* self.radius self.radius))))
```

A method whose first argument is `self` is an instance method. Call every method in the form `TypeName/function-name`.

```lisp
(let c (Circle/new (Point 0.0f 0.0f) 5.0f))
(println! "area = {}" (Circle/area c))
(println! "contains? = {}" (Circle/contains? c (Point 1.0f 1.0f)))
```
