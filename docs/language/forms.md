# Definitions and Expressions

A Singulisp program is a sequence of S-expressions. After macro expansion, the forms produced by the reader are lowered into a list of top-level definitions. Runtime expressions must appear in the body of a function or similar definition; they cannot appear directly at the top level. This page summarizes the forms accepted by lowering.

## Top-Level Definitions

The implementation recognizes the following definition forms.

| Form | Purpose |
|---|---|
| `defn` | Function |
| `const-fn` | Pure function callable at compile time |
| `const-gen` | Typed code-generation function |
| `defstruct` | Struct |
| `deftype` | Algebraic data type |
| `deftrait` | Trait |
| `impl` | Trait implementation |
| `definterface` | ABI-boundary interface |
| `deftype-fn` | Type-level function |
| `defregion` | Region contract |
| `extern` | External ABI declaration |
| `use` | Module import / re-export |
| `const` | Compile-time constant |
| `deftest` | Test |
| `defbench` | Benchmark |
| `defmacro` | Macro |

An attribute section, `#[...]`, modifies the immediately following definition or corresponding expression. See [Attributes](attributes.md) for the available attributes and their targets.

## Functions

```lisp
(defn add [(a : i32) (b : i32)] -> i32
  (+ a b))

(defn identity [T] [(x : T)] -> T
  x)

(defn first [(N : usize)] [(xs : [Array i32 N])] -> i32
  (Array/get xs 0))
```

Type parameters go in the first `[...]` after the name, and value parameters go in the following `[...]`. A const generic parameter has the form `(N : usize)`. `usize` is not a runtime integer type. Specify the return type with `-> Type`. See [Functions](functions.md) and [Generics and Traits](traits.md) for details.

Anonymous functions use `fn`.

```lisp
(let twice (fn [(x : i32)] -> i32 (* x 2)))
(twice 21)
```

## Value Bindings and Mutation

```lisp
(let x 42)
(let y : i64 42)
(let mut count 0)
(set! count (+ count 1))
```

`let` creates an immutable binding, while `let mut` creates a mutable binding. The left-hand side of `set!` must be an expression that lowering recognizes as a place, such as a mutable local, dereference, field, or index.

## Conditionals

`if` always requires an else expression, and both branches must unify to the same result type.

```lisp
(if (> x 0)
  x
  (- 0 x))
```

`when` and `unless` provide conditional sequential execution. `cond` evaluates its clauses from top to bottom.

```lisp
(when ready
  (start-work))

(cond
  [(< x 0) -1]
  [(== x 0) 0]
  [:else 1])
```

The logical forms are `not`, `and`, and `or`. `and` and `or` use short-circuit evaluation.

## Pattern Matching

```lisp
(match value
  [(Some x) x]
  [None 0])
```

Each arm has the form `[pattern expression]`. See [Control Flow and Pattern Matching](control-flow.md) for guards, exhaustiveness, and the rules for destructuring owned values.

## Sequencing and Iteration

`do` evaluates expressions from left to right and returns the final value.

```lisp
(do
  (step-a)
  (step-b)
  result)
```

Low-level iteration uses `loop` and `recur`. `break` and `continue` are also available.

```lisp
(loop [(i : i32 0) (sum : i32 0)]
  (if (>= i 10)
    sum
    (recur (+ i 1) (+ sum i))))
```

`for` iterates over ranges, `Vec`, and concrete types that implement the well-known `Iterable` protocol. `while` executes its body while its condition is true.

## References and unsafe

```lisp
(let shared (& value))
(let exclusive (& mut value))
(let copied (* shared))
```

`&` and `& mut` create borrows, and unary `*` dereferences them. Borrowing follows the rules in [Ownership and Borrowing](ownership.md). `unsafe` is a boundary for low-level operations explicitly permitted by the implementation. It lowers transparently into its body and does not relax type, ownership, or borrow checking.

## Error Propagation

`?` unwraps a `Result` and immediately returns from the current function if it is `Err`.

```lisp
(defn load [] -> [Result i32 String]
  (let n (? (parse-number)))
  (Ok n))
```

See [Option / Result and Error Propagation](error-handling.md) for details.

## Regions and Allocation

- `(@R handle body...)` — scope of a region instance
- `arena/alloc!` — infallible allocation
- `arena/try-alloc` — fallible allocation
- `arena/reset!` — resets a reusable arena
- `persist`, `thaw`, `freeze` — boundaries between mutable and persistent representations

See [Regions and Allocation](regions.md) for combinations of definitions and attributes.

## Concurrency, SPMD, and GPU

- structured concurrency: `with-scope`, `scope/spawn`, `sync`
- channel: `channel/new`, `channel/send`, `channel/recv`, `channel/close`
- SPMD: `spmd`, `spmd/tile`, `spmd/window`, `spmd/static-partition`
- GPU: `gpu/dispatch`, `gpu/dispatch-async`, `await`

Their constraints are documented in [Concurrency](concurrency.md), [SPMD](../performance/spmd.md), and [GPU](../performance/gpu.md), respectively.

## Typed Staging

`expr` constructs an `[Expr T]`, and `unquote` embeds a staged expression. See [Compile-Time Computation](compile-time.md) for compile-time computation with `const-fn` and `const-gen`.

## Function Application and Evaluation Order

A list whose head is not a special form is a function application.

```lisp
(function arg1 arg2)
```

The callee is resolved statically from the leading symbol and is not evaluated as a value. Arguments are evaluated from left to right. In code that depends on side effects, make intermediate results explicit with `let` or `do` instead of obscuring the evaluation order.
