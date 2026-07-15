# Built-in APIs

This page indexes APIs that the compiler recognizes and lowers directly. See
[Standard Modules](modules.md) for APIs implemented in Singulisp source.

## Scalar Arithmetic

```lisp
(+ a b)
(- a b)
(* a b)
(/ a b)
(% a b)
(== a b)
(!= a b)
(< a b)
(<= a b)
(> a b)
(>= a b)
```

The ordering comparison operators are defined only for numeric types. Logical
operations are `not`, `and`, and `or`; bitwise operations are `bit-not`, `bit-and`,
`bit-or`, `bit-xor`, `bit-shl`, and `bit-shr`.

## Numeric Conversion

Use a `to-T` form, from `to-i8` through `to-f64`, or the corresponding `T/from` form.

- Integer to integer: narrowing truncates and preserves the low bits. Widening
  sign-extends a signed source and zero-extends an unsigned source. A same-width
  signedness conversion preserves the bit pattern.
- Integer to float: performs a numeric conversion according to the source's
  signedness.
- Float to integer: a saturating conversion. NaN becomes zero, and out-of-range
  values become the target type's minimum or maximum.
- `f32` ↔ `f64`: widening or narrowing conversion.

```lisp
(to-i32 3.7f)
(to-u64 42)
(f32/from 10)
```

## Integer Intrinsics

`popcount`, `clz`, `ctz`, `bswap`, `reverse-bits`, `rotl`, and `rotr` are available
for `i32`, `i64`, `u32`, and `u64`. `sat-add` and `sat-sub` are available for every
8-, 16-, 32-, and 64-bit signed and unsigned integer type.

```lisp
(u32/popcount value)
(u64/rotl value amount)
(i16/sat-add a b)
```

The following operations are also provided for high-performance kernels.

- `i32/select-branchless`
- 32- and 64-bit types: `funnel-shl`, `funnel-shr`, `mul-hi`, `mul-wide`
- `u32`, `u64`: `clmul-wide`
- `u32/crc32c-u8`, `crc32c-u16`, `crc32c-u32`, `crc32c-u64`
- `u32`, `u64`: `add-carry`, `sub-borrow`

Wide and carry operations return the built-in records `U64Wide`, `I64Wide`,
`U64Carry`, `U64Borrow`, `U32Carry`, and `U32Borrow`.

## Floating-Point and Math

See [Math Functions](math.md) for `f32/*` and `f64/*` constants, unary and binary
math functions, and floating-point modes. First-class vector, quaternion, and matrix
APIs are documented under [Built-in Math Types](../language/math-types.md).

## SIMD

The six named SIMD types are:

`f32x2`, `f32x4`, `f64x2`, `i32x2`, `i32x4`, `u32x4`

The main APIs include `simd/add`, `sub`, `mul`, `div`, `min`, `max`, `dot`, `extract`,
`insert`, `shuffle`, `splat`, comparisons, `select`, and unary math functions. See
[SIMD](../performance/simd.md) for details.

## String and StrSlice

Main low-level APIs:

- `String/len`, `concat`, `contains`, `starts-with`, `ends-with`, `index-of`
- `String/substring`, `String/slice`, `String/from`
- `String/byte-at`, `String/char-at`, `String/of-char`, `String/push!`
- `String/to-upper`, `to-lower`, `trim`, `replace`, `split`
- `String/parse-i64`, `String/parse-f64`, `String/f64-to-string`
- `String/eq`, `String/lt`

See [Strings](strings.md) for ownership, SSO, byte semantics, and the differences in
bounds checking.

## Formatting and Output

```lisp
(format "{}:{}" name value)   ;; Returns a String
(println! "{}:{}" name value) ;; stdout + newline
```

`format` can be called from an ordinary function and returns an owned `String`.
Results of 15 bytes or fewer fit in SSO; longer results allocate on the heap as needed.
`println!` requires an external or diagnostics context. Format arguments may be any
integer width, `f32`, `f64`, `bool`, `String`, or `StrSlice`.

## Assertions

```lisp
(assert! condition)
(assert-eq! left right)
(assert-ne! left right)
```

In addition to numeric values, `assert-eq!` and `assert-ne!` accept values of the same
`bool` or `String` type. Failure is a runtime trap and cannot be recovered as an
exception. The surface language has no null-pointer literal; test an address with an
expression such as `(== address (to-u64 0))`.

## Collections and Allocation

`Box`, `Vec`, `Array`, SoA collections, and persistent collections are compiler
built-in types and operations. See [Collections](collections.md) for their APIs,
[Ownership](../language/ownership.md) for ownership semantics, and
[Regions](../language/regions.md) for arena policies.

## I/O, Concurrency, and GPU

- I/O, files, streams, and processes: [Input and Output](io.md)
- `with-scope`, `scope/spawn`, `sync`, and `ChannelHandle`:
  [Structured Concurrency](../language/concurrency.md)
- `spmd`: [SPMD](../performance/spmd.md)
- `GpuBuffer`, dispatch, and shader operations: [GPU](../performance/gpu.md)

## Dynamic Modules

The dynamic-module built-in is `module/load`.

```lisp
(module/load region-handle path Interface host-services) ;; -> ModuleHandle
```

There are no `module/call` or `module/unload` surface built-ins. Call interface methods
as `Interface/method`; the handle lifetime follows its registered region. See
[FFI](../platform/ffi.md) for a complete example.

## Names That Cannot Be Redefined

The following type names cannot be redefined with `defstruct` or `deftype`:

- Scalar/runtime: `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`,
  `f64`, `bool`, `Bool`, `String`, `Reader`, `Writer`, `AllocError`
- Integer result records: `U64Wide`, `I64Wide`, `U64Carry`, `U64Borrow`, `U32Carry`,
  `U32Borrow`
- SIMD: the six types listed above
- Constructors: `Result`, `Option`, `Array`, `Vec`, `Box`, `PVec`, `PMap`, `GpuBuffer`,
  `Expr`, `Fn`
- Math: `Vec2`, `Vec3`, `Vec4`, `Quat`, `Float3`, `Mat3`, `Mat3r`, `Mat4`, `Mat4r`,
  `Mat3x4`

Reserved variants are `Ok`, `Err`, `Some`, `None`, and `AllocOom`. Reserved function
and form names are `println!`, `assert!`, `assert-eq!`, and `assert-ne!`.

## Reserved Words

The following words cannot be used as variable or parameter names.

- Definitions: `defn`, `defstruct`, `deftype`, `defmacro`, `deftrait`, `deftype-fn`,
  `defregion`, `impl`, `extern`, `const`, `const-fn`, `const-gen`, `use`
- Control flow: `let`, `set!`, `if`, `do`, `match`, `loop`, `recur`, `fn`, `when`,
  `unless`, `cond`, `for`, `while`
- Logic and literals: `and`, `or`, `not`, `true`, `false`, `nil`
- Ownership and parallelism: `ref`, `mut`, `unsafe`, `with-scope`, `sync`, `spmd`
- Implemented control words: `await`, `break`, `continue`
- Reserved (no form): `import`, `export`, `module`, `trait`, `type`, `struct`, `enum`,
  `async`, `yield`, `return`, `where`, `as`, `in`, `of`

The compiler also recognizes `StrSlice`, the handle types, SoA types, `DeserError`,
and others. The list above contains only the type names that the reserved-name checker
explicitly rejects when redefined.
