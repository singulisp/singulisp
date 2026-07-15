# Macros

Singulisp's macro system generates and transforms code at compile time. It builds CST templates from S-expressions using quasiquote and unquote.

## Position in the Pipeline

Macro expansion takes place before the Lower phase (CST-to-AST conversion).

```
Source → Reader (CST) → MacroExpander (CST→CST) → Lower (CST→AST) → type checking …
```

Because macros consume and produce CST (concrete syntax tree) nodes, they **have no type information**. Type checking occurs in the Lower phase after macro expansion.

## `defmacro` Syntax

Define a macro with `defmacro`. A macro is a transformation rule that accepts S-expressions and returns a new S-expression. `defmacro` may appear only at the top level; it cannot be defined inside a function body.

```lisp
(defmacro macro-name [parameter ...]
  transformed-expression)
```

Macro expansion proceeds outermost-first. Exceeding **65,536 expansion steps** produces an **E-MACRO-0901** error, detecting cyclic infinite expansion.

### Basic Macros

```lisp
(defmacro when-do [condition & body]
  `(if ~condition (do ~@body) ()))

(defmacro unless-do [condition & body]
  `(if (not ~condition) (do ~@body) ()))
```

The `when-do` macro evaluates its body only when the condition is `true`. `unless-do` does the opposite.

### Example

```lisp
(when-do (> health 0)
  (update-physics entity)
  (render entity))

;; After expansion:
;; (if (> health 0) (do (update-physics entity) (render entity)) ())
```

## quasiquote --- Building Macro Templates

### `` `expr `` --- Backquote (Quasiquote)

A backquote creates a macro template in which `~` (unquote) and `~@` (unquote-splicing) may be used. It is used in macro definitions.

```lisp
`(+ ~a ~b)  ;; Generate a list with the values of a and b inserted
```

The reader desugars `'x` to `(quote x)`, but `quote` is neither a runtime expression nor a data constructor. Consequently, `'x` and `(quote x)` cannot be used in ordinary code; use backquote for code generation.

## unquote --- Evaluating an Expression Inside a Backquoted Block

`~expr` evaluates an expression inside a backquoted block and inserts its result.

```lisp
(defmacro double-it [x]
  `(+ ~x ~x))

(double-it 5)
;; After expansion: (+ 5 5)
;; Result: 10
```

Without `~`, `x` is emitted unchanged as a symbol. With `~`, the value bound to `x`—the expression passed as an argument—is inserted.

## unquote-splicing --- Splicing a List

`~@expr` expands the elements of a list and inserts them into the containing list.

```lisp
(defmacro my-do [& items]
  `(do ~@items))

(my-do (let x 1) (let y 2) (+ x y))
;; After expansion: (do (let x 1) (let y 2) (+ x y))
```

Using `~items` without `~@` nests `items` as a list. With `~@`, the list contents are spliced as individual elements.

### Difference Between `~` and `~@`

```lisp
;; When items = (1 2 3)

`(do ~items)    ;; => (do (1 2 3))   ← the list is nested
`(do ~@items)   ;; => (do 1 2 3)     ← the elements are spliced
```

## Variadic Parameters

An `& rest` parameter allows a macro to accept any number of arguments. The parameter following `&` is bound to a list containing all remaining arguments.

```lisp
(defmacro begin [& exprs]
  `(do ~@exprs))

(begin
  (let x 1)
  (let y 2)
  (+ x y))
;; After expansion: (do (let x 1) (let y 2) (+ x y))
```

### Combining Fixed and Variadic Parameters

```lisp
(defmacro with-log [label & body]
  `(do
    (println! "[{}] start" ~label)
    ~@body
    (println! "[{}] end" ~label)))

(with-log "update"
  (update-physics world)
  (update-ai world))
```

## Hygienic Macros

Singulisp macros are partially hygienic. Variable names introduced inside a macro do not collide with variable names at the macro's call site.

```lisp
(defmacro swap! [a b]
  `(do
    (let __tmp ~a)
    (set! ~a ~b)
    (set! ~b __tmp)))
```

`__tmp` is a temporary variable inside the macro and does not affect the caller's scope.

Alpha conversion applies to the following binding positions introduced directly by the macro within its template:

- binding names in `let` and `let mut`;
- pattern binding names in `match`, excluding variant names, literals, and `_`;
- variable names in the binding vector of `loop`;
- variable names in the binding vector of `for`; and
- parameter names of `fn` and `defn`.

The interior of an expression interpolated from the call site with `~` or `~@` is not collected for alpha conversion by the macro. To place a caller-supplied symbol explicitly in a binding position, interpolate it as in `~name`. There is no dedicated escape hatch for intentionally exposing a fixed, uninterpolated name at the call site, as an anaphoric macro might do.

```lisp
(let mut x 10)
(let mut y 20)
(swap! x y)
;; x = 20, y = 10
```

## Compiler-provided Syntax

### Special Forms (Compiled Directly to IR Nodes)

`println!`, `assert!`, `assert-eq!`, and `assert-ne!` are not user-defined macros. They are **special forms** that the compiler compiles directly to IR nodes. Although their trailing `!` notation resembles `defmacro`, the macro expander does not process them.

| Special form | Description | Example |
|----------|------|-------|
| `println!` | Formatted standard output; requires an `#[external]` or `#[diagnostics]` context | `(println! "x = {}" x)` |
| `assert!` | Verifies a condition; may be used in pure functions | `(assert! (> x 0))` |
| `assert-eq!` | Verifies equality; may be used in pure functions | `(assert-eq! x 42)` |
| `assert-ne!` | Verifies inequality; may be used in pure functions | `(assert-ne! x 0)` |

Because `println!` has an I/O side effect, calling it from a pure function is a compile-time error. The `assert!` family is treated as runtime checking that aborts on failure and is not subject to effect-system restrictions.

### Conditional Special Forms

`when` and `unless` are not `defmacro` definitions pre-registered in the macro environment. They are special forms that lowering transforms directly into `if` forms.

| Special form | Equivalent form |
|--------|--------|
| `(when cond body...)` | `(if cond (do body...) ())` |
| `(unless cond body...)` | `(if (not cond) (do body...) ())` |

A trailing `!` conventionally indicates a macro or special form, but the language specification does not enforce this convention.

## Macro Examples

### Simplifying Conditional Branching

```lisp
(defmacro when-let [name expr & body]
  `(match ~expr
    [(Some ~name) (do ~@body)]
    [None ()]))

;; Usage
(when-let player (find-player id)
  (update-player player)
  (render-player player))
```

### Abstracting a Repetition Pattern

```lisp
(defmacro dotimes [var count & body]
  `(loop [(~var : i32 0)]
    (if (< ~var ~count)
      (do
        ~@body
        (recur (+ ~var 1)))
      ())))

;; Usage
(dotimes i 10
  (println! "i = {}" i))
```

### Resource Management

```lisp
(defmacro with-resource [name init cleanup & body]
  `(do
    (let ~name ~init)
    (let __result (do ~@body))
    ~cleanup
    __result))

;; Example
(with-resource fh (open-file "data.txt") (close-file fh)
  (process fh))
```

## Guidelines for Defining Macros

### Evaluate Arguments Only Once

Using a macro argument multiple times evaluates the argument expression multiple times. This is a problem when the expression has side effects.

```lisp
;; Bad example: x is evaluated twice
(defmacro double-bad [x]
  `(+ ~x ~x))

;; Good example: evaluate once through a temporary variable
(defmacro double-good [x]
  `(do (let __val ~x)
    (+ __val __val)))
```

### Prefer Functions to Macros

Macros transform code at compile time and can make debugging harder. Prefer an ordinary function whenever one can express the operation. Use a macro only when syntax extension or delayed evaluation is required.

---

## Control Structures Within Templates

The following compile-time control structures may be used in a macro body during the template-evaluation phase. They are evaluated during macro expansion, not at runtime.

### `(if cond then else)` --- Conditional Branching

```lisp
(defmacro emit-op [op a b]
  (if (== op :add)
    `(+ ~a ~b)
    `(* ~a ~b)))
```

When `cond` is the Boolean `false` or the empty list `()`, `else` is selected. Otherwise, `then` is selected.

### `(let [name val] body)` --- Local Bindings

```lisp
(defmacro unless [cond & body]
  (let [negated `(not ~cond)]
    `(if ~negated (do ~@body) ())))
```

### `(empty? expr)` / `(first expr)` / `(rest expr)` --- List Operations

```lisp
(defmacro sum-all [& nums]
  (if (empty? nums)
    `0
    `(+ ~(first nums) (sum-all ~@(rest nums)))))
```

### `(== a b)` --- Equality Comparison

Symbols, keywords, integers, Booleans, and strings can be compared. Values of different types compare as `false`.

---

## Error Codes

| Code | Description |
|--------|------|
| `E-MACRO-0001` | Call to an undefined macro |
| `E-MACRO-0002` | Argument-count mismatch (too many or too few) |
| `E-MACRO-0003` | Invalid `defmacro` syntax |
| `E-MACRO-0901` | Expansion-step limit (65,536) exceeded |
| `E-MACRO-0902` | `~@` used outside a list context |
| `E-MACRO-0903` | `~` or `~@` used outside quasiquote |
