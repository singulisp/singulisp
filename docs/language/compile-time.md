# const-fn and Typed Staging

Singulisp provides two features for compile-time computation. `const-fn` defines functions evaluated at compile time, while typed staging (const-gen) is an advanced facility that generates code at compile time.

## const-fn

### Basics

Functions defined with the `const-fn` keyword are evaluated at compile time. They are parsed with exactly the same syntax as `defn` and treated as pure functions that can be evaluated at compile time. They may be called from a `const` initializer.

```lisp
(const-fn square [(n : i32)] -> i32
  (* n n))

(const SIZE : i32 (square 16))  ; Evaluated to 256 at compile time
```

After compilation, `SIZE` is embedded as the constant `256` in the runtime program.

The `#[external]` attribute is forbidden on a const-fn (**E-LOWER-0130**). A const-fn must always be pure.

### Permitted Operations

The following operations are available within a const-fn.

| Operation | Example |
|---|---|
| Arithmetic | `+`, `-`, `*`, `/`, `%` |
| Comparisons | `<`, `>`, `<=`, `>=`, `==`, `!=` |
| Logical operations | `and`, `or`, `not` |
| Conditionals | `if` |
| Calls to another const-fn | `(other-const-fn arg)` |
| Fixed-size array literals | `(array 1 2 3 4)` |
| Fixed-size array fill | `(Array/fill 256 0)` |

Operations with side effects, including I/O, mutable references, and heap allocation, are not available.

### Examples

#### Compile-Time Table Generation

```lisp
(const-fn compute-lut-entry [(i : i32)] -> i32
  (* i i))

;; Compute each lookup-table entry at compile time
(const LUT_0 : i32 (compute-lut-entry 0))
(const LUT_1 : i32 (compute-lut-entry 1))
(const LUT_2 : i32 (compute-lut-entry 2))
;; ...
```

#### Computing Configuration Values

```lisp
(const-fn next-power-of-two [(n : i32)] -> i32
  (if (<= n 1)
    1
    (* 2 (next-power-of-two (/ (+ n 1) 2)))))

(const BUFFER_SIZE : i32 (next-power-of-two 1000))  ; 1024
```

### const Declarations

`const` declares a compile-time constant. Its value may be a literal, a const-fn call, or a fixed-size array literal or fill operation.

```lisp
(const PI : f64 3.141592653589793)
(const TAU : f64 (* 2.0 PI))
(const MAX_ENTITIES : i32 1024)
(const CRC_TABLE : [Array u32 4] (array 0 1 7 9))
(const ZEROS : [Array u8 16] (Array/fill 16 0))
```

## Typed Staging (const-gen)

Typed staging is an advanced metaprogramming facility for generating program fragments at compile time. A `const-gen` is parsed with the same syntax as `defn` and has the `is_const_gen` flag set. Its return type must be `[Expr T]` (**E-STAGE-0001**). It cannot have the `#[external]` attribute.

### The `Expr<T>` Type (`[Expr T]`)

`[Expr T]` represents a code fragment constructed at compile time. `T` is the type of the value that the generated code returns at runtime.

### Constraints and Error Codes

| Code | Description |
|--------|------|
| `E-STAGE-0001` | The const-gen return type is not `[Expr T]` |
| `E-STAGE-0002` | The expansion recursion limit of 128 was exceeded |
| `E-STAGE-0003` | The operand of `~` is not a permitted variable reference or const-gen call |
| `E-STAGE-0004` | `expr` was used outside a const-gen body |

### `expr` — Constructing Code

The `expr` form may be used **only within a const-gen body** (otherwise E-STAGE-0004). It constructs a code fragment.

```lisp
(const-gen make-add-one [] -> [Expr i32]
  (expr (+ 1 1)))
```

### `~` — Splicing (Embedding Expressions)

The `~` operator embeds a compile-time value or `Expr<T>` in another code fragment. Within `(expr ...)`, it may operate on a const-gen parameter, a local variable reference, or a const-gen call. Any other operand produces E-STAGE-0003. At the top level, `~` in a const initializer accepts only a const-gen call.

```lisp
(const-gen make-double [(x : [Expr i32])] -> [Expr i32]
  (expr (+ ~x ~x)))
```

`~x` expands the code fragment held by `x` at that position. Expansion has a recursion limit of 128; exceeding it reports E-STAGE-0002.

### Recursive Code-Expansion Example

```lisp
(const-gen make-sum [(n : i32)] -> [Expr i32]
  (if (== n 0)
    (expr 0)
    (expr (+ ~n ~(make-sum (- n 1))))))

(const S5 : i32 ~(make-sum 5))
```

This function recursively generates an expression of the form `(+ n (+ (n-1) (+ ... 0)))`. For `n = 5`, it generates code equivalent to the following:

```lisp
(+ 5 (+ 4 (+ 3 (+ 2 (+ 1 0)))))  ; => 15
```

### Type Safety

As its name suggests, typed staging preserves type information. Attempting to splice an `Expr<i32>` where an `Expr<f32>` is expected is a compile-time error. This guarantees the type safety of generated code at code-generation time.

```lisp
(const-gen bad-splice [] -> [Expr f32]
  (let x : [Expr i32] (expr 42))
  (expr (+ 1.0f ~x)))  ; Compile-time error: Expr<i32> cannot be embedded in Expr<f32>
```

### const-fn Compared with Typed Staging

| Characteristic | const-fn | Typed staging |
|---|---|---|
| Purpose | Compute values at compile time | Generate code at compile time |
| Return value | Ordinary value (i32, f32, etc.) | `Expr<T>` (code fragment) |
| Complexity | Simple computation | Metaprogramming |
| Uses | Constant tables and configuration values | Loop unrolling and specialized code generation |
