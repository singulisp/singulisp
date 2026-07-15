# REPL (Interactive Development Environment)

The Singulisp REPL is an interactive development environment backed by LLVM JIT. It evaluates expressions, defines functions, and inspects types in real time.

---

## Starting the REPL

```bash
gu-cli repl
```

The following banner appears at startup:

```
Singulisp REPL (LLVM JIT)
Type :quit to exit, :help for help
```

Input prompt: `singulisp>`

Input proceeds through lexical analysis, S-expression reading, macro expansion, and classification. Function definitions and `let` bindings accumulate across the session.

---

## Command Reference

The following commands are available within the REPL.

| Command | Description |
|---------|-------------|
| `:quit` / `:q` | Exit the REPL |
| `:help` / `:h` | Display help |
| `:reset` | Reset the environment and discard all definitions |
| `:clear` | Clear the screen |
| `:defs` | List defined functions |
| `:type EXPR` / `:t EXPR` | Display the type of an expression |

---

## Basic Usage

### Evaluating Expressions

```
singulisp> (+ 1 2)
  [1] frontend: 0.03ms | mir: 0.01ms | llvm-jit: 0.12ms | total: 0.16ms
3 : i32
singulisp> (* 3.14 2.0)
  [2] frontend: 0.02ms | mir: 0.01ms | llvm-jit: 0.08ms | total: 0.11ms
6.28 : f64
singulisp> (to-u8 255)
  [3] frontend: 0.02ms | mir: 0.01ms | llvm-jit: 0.08ms | total: 0.11ms
255 : u8
```

### Defining and Calling Functions

```
singulisp> (defn square [(n : i32)] -> i32 (* n n))
square : [Fn [i32] -> i32]
singulisp> (square 7)
  [1] frontend: 0.03ms | mir: 0.01ms | llvm-jit: 0.15ms | total: 0.19ms
49 : i32
singulisp> (square 12)
  [2] frontend: 0.02ms | mir: 0.01ms | llvm-jit: 0.08ms | total: 0.11ms
144 : i32
```

### Inspecting Defined Functions

```
singulisp> :defs
  square : [Fn [i32] -> i32]
```

### Structs and Pattern Matching

```
singulisp> (defstruct Point (x : f32) (y : f32))
struct Point
singulisp> (let p (Point 3.0f 4.0f))
  [3] frontend: 0.04ms | mir: 0.02ms | llvm-jit: 0.20ms | total: 0.26ms
<Point @ 0x7f...> : Point
singulisp> (f32/sqrt (+ (* p.x p.x) (* p.y p.y)))
  [4] frontend: 0.03ms | mir: 0.01ms | llvm-jit: 0.15ms | total: 0.19ms
5.0 : f32
singulisp> (deftype Shape (Circle f32) (Rect f32 f32))
type Shape
singulisp> (match (Circle 5.0f)
  ...>   [(Circle r) (* 3.14159f (* r r))]
  ...>   [(Rect w h) (* w h)])
  [5] frontend: 0.05ms | mir: 0.02ms | llvm-jit: 0.25ms | total: 0.32ms
78.53975 : f32
```

For multiline input, the `...>` prompt continues accepting input until all parentheses are closed.

---

## Input Classification

Input forms are classified into the following six categories by the leading symbol of the top-level form.

| Category | Matching Pattern | Processing |
|----------|------------------|------------|
| FnDef | `(defn name ...)` | Adds the definition to the session and displays its type signature |
| StructDef | `(defstruct name ...)` | Collects and stores the struct name and field information |
| TypeDef | `(deftype name ...)` | Registers and stores ADT variant information |
| MacroDef | `(defmacro name ...)` | Adds the macro definition to the session |
| LetBinding | `(let name expr)` / `(let mut name expr)` | Evaluates the right-hand side, displays the result, and adds the binding to the session |
| Expression | Anything else | JIT-compiles and immediately evaluates the expression, then displays its value and type |

## Expression-Evaluation Pipeline

Expressions entered in the REPL pass through the following pipeline:

```
Input → CST → AST → IR → MIR (O1 optimization) → LLVM JIT → Execution
```

1. Construct an evaluation wrapper function containing the accumulated `let` bindings and the input expression.
2. Convert all accumulated definitions and the evaluation function from CST to AST to IR.
3. Determine the expression's actual return type and finalize the evaluation function's signature.
4. Apply O1 optimization to MIR.
5. JIT-compile with LLVM and call the function.

After evaluation, the elapsed time for each phase is written to stderr.

---

## Compilation-Time Display

When an expression or the right-hand side of a `let` binding is evaluated, the time spent in each compilation phase is written to stderr. No timing is displayed when defining a `defn`, `defstruct`, `deftype`, or `defmacro`.

```
singulisp> (defn fib [(n : i32)] -> i32
  ...>   (if (<= n 1) n (+ (fib (- n 1)) (fib (- n 2)))))
fib : [Fn [i32] -> i32]
singulisp> (fib 10)
  [1] frontend: 0.05ms | mir: 0.02ms | llvm-jit: 0.31ms | total: 0.38ms
55 : i32
```

This helps identify compilation bottlenecks quickly.

---

## Line Editing and History

The REPL is based on the rustyline library and provides the following line-editing features.

| Key | Description |
|-----|-------------|
| `Ctrl-R` | Search backward through history |
| `Ctrl-C` | Cancel the current input |
| `Ctrl-D` | Exit the REPL when the input line is empty |
| `↑` / `↓` | Move backward or forward through history |
| `Ctrl-A` / `Ctrl-E` | Move to the beginning or end of the line |
| `Ctrl-W` | Delete the preceding word |

### Persistent History

Input history is saved automatically to `~/.gu/history`. Previous input remains available after restarting the REPL.

---

## Resetting the Environment

Use `:reset` to discard all defined functions and variables and return to the initial state.

```
singulisp> (defn add [(a : i32) (b : i32)] -> i32 (+ a b))
add : [Fn [i32 i32] -> i32]
singulisp> :defs
  add : [Fn [i32 i32] -> i32]
singulisp> :reset
REPL state has been reset.
singulisp> :defs
(no definitions)
```

---

## Limitations

- Modules cannot be loaded with file-based `use` inside the REPL
- GPU execution of functions with the `#[shader]` attribute is not supported
- Because the REPL uses an O1 MIR pipeline and a maximum SIMD width of 128 bits, its optimization conditions and generated code are not necessarily identical to an AOT release build
