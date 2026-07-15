# Runtime Utilities

This page summarizes the main built-in utilities that do not belong to a module. See
[Standard Modules](modules.md) for the list of standard `std` modules.

## Time

```lisp
(time/now-ms)  ;; -> i64
```

Returns the number of milliseconds elapsed since the Unix epoch. Because it reads
external state, this operation requires an external context.

```lisp
#[external]
(defn measure [(work : [Fn [] -> ()])] -> i64
  (let start (time/now-ms))
  (work)
  (- (time/now-ms) start))
```

## Process Termination

```lisp
(process/exit code)
```

Immediately terminates the process with the specified exit code. It requires an
external context and performs neither a normal return nor stack unwinding.

## Assertions

| Form | Condition |
|---|---|
| `assert! condition` | Traps if `condition` is `false` |
| `assert-eq! left right` | Traps if the values are not equal |
| `assert-ne! left right` | Traps if the values are equal |

```lisp
(defn divide [(a : f32) (b : f32)] -> f32
  (assert! (!= b 0.0f))
  (/ a b))
```

An assertion failure aborts the process. It cannot be recovered as an exception and
does not unwind the stack.

## Numeric Conversion

Explicit conversions use the `to-T` form. `T/from` is the corresponding alias.

```lisp
(let i (to-i32 3.7f))
(let f (to-f64 42))
(let u (u64/from 100))
```

The supported target types are `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`,
`f32`, and `f64`. Floating-point-to-integer conversion is saturating: NaN becomes
zero, and out-of-range values are clamped to the target type's minimum or maximum.

## Bit Operations

General-purpose bitwise operations:

```lisp
(bit-and a b)
(bit-or a b)
(bit-xor a b)
(bit-not a)
(bit-shl value amount)
(bit-shr value amount)
```

`^`, `&`, and `|` cannot be used as bitwise operators in expression position. Shift
amounts are masked to the width of the target integer type.

Integer type namespaces provide the following operations.

| Operation | Meaning |
|---|---|
| `T/popcount` | Number of set bits |
| `T/clz`, `T/ctz` | Number of leading or trailing zero bits |
| `T/bswap` | Reverses byte order |
| `T/reverse-bits` | Reverses bit order |
| `T/rotl`, `T/rotr` | Rotation |
| `T/sat-add`, `T/sat-sub` | Saturating addition or subtraction |

```lisp
(i32/popcount 202)  ;; 0b11001010 -> 4
(u32/clz 1)         ;; 31
(u8/sat-add 250 10) ;; 255
```

Numeric literals cannot contain `_` separators. For example, `1_000` is invalid;
write `1000` instead.

## Arithmetic Overflow

In a normal build, integer arithmetic is lowered to the target's integer operations.
Use the CLI option `--overflow-trap` to trap on addition, subtraction, or
multiplication overflow in a debug build.

```bash
gu-cli run src/main.gu --debug --overflow-trap
```

When arithmetic must always wrap explicitly, use `wrap-add`, `wrap-sub`, and
`wrap-mul` from `std::num`.
