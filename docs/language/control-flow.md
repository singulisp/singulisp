# Control Flow

All Singulisp control-flow constructs are **expressions** and return values. `if` can be used like a ternary operator, `match` handles complex branches exhaustively, and `loop`/`recur` provides efficient iteration.

---

## if Expressions

`if` performs conditional branching and returns its result as a value.

### Syntax

```lisp
(if condition then-expression else-expression)
```

### Basic Examples

```lisp
(let result (if (> x 0) "positive" "non-positive"))
```

```lisp
(defn abs [(x : i32)] -> i32
  (if (< x 0) (- 0 x) x))
```

### Nested if

```lisp
(defn classify [(x : i32)] -> String
  (if (> x 0)
    "positive"
    (if (< x 0)
      "negative"
      "zero")))
```

### Rules

- The condition must have type `bool`. Integers and references are not converted implicitly.
- The then-expression and else-expression must have the same type.
- The else-expression cannot be omitted; use `when` instead.

---

## do Blocks

`do` evaluates multiple expressions sequentially and returns the value of the final expression as the value of the entire block.

### Syntax

```lisp
(do
  expression1
  expression2
  ...
  final-expression)
```

### Basic Example

```lisp
(let result
  (do
    (println! "step 1")
    (println! "step 2")
    42))
; result is 42
```

### Order of Side Effects

Expressions within `do` are evaluated from top to bottom. Side effects, such as output and assignment, occur in that order.

```lisp
#[external]
(defn initialize [] -> i32
  (do
    (println! "initializing...")
    (let config (fs/read-to-string "config.txt"))
    (println! "config loaded")
    0))
```

---

## match Expressions

`match` selects among patterns for ADTs defined with `deftype`, vector-form tuples, integers, booleans, strings, and character literals. There is no pattern that directly destructures a struct (`defstruct`).

### Syntax

```lisp
(match scrutinee
  [pattern1 expression1]
  [pattern2 expression2]
  ...)
```

Integer, boolean, string, and character values can be selected with literal patterns, as described below.

### Wildcards

```lisp
(match opt
  [_ "anything"])
```

The underscore `_` matches every value and creates no binding.

### Enum Patterns

```lisp
(deftype Shape
  (Circle f32)
  (Rect f32 f32)
  (Point))

(match shape
  [(Circle r) (* 3.14159f (* r r))]
  [(Rect w h) (* w h)]
  [Point 0.0f])
```

Each pattern matches an enum variant. A variant with arguments is enclosed in parentheses; a variant without arguments is written as-is.

### Option/Result Patterns

```lisp
(match opt
  [(Some value) (println! "found: {}" value)]
  [None         (println! "not found")])

(match result
  [(Ok val) val]
  [(Err e)  (do (println! "error: {}" e) 0)])
```

### Literal Patterns

Integer, boolean, string, and character literals may be used as patterns. The compiler desugars them into an `if-else` chain.

```lisp
(match code
  [200 "OK"]
  [404 "Not Found"]
  [500 "Internal Server Error"]
  [_   "Unknown"])

(match flag
  [true  "enabled"]
  [false "disabled"])
```

### Nested Variant Patterns

When a variant field contains another ADT, nested patterns can destructure the deep structure in a single `match`.

```lisp
(defn flatten [(opt : [Option [Option i32]])] -> [Option i32]
  (match opt
    [(Some (Some v)) (Some v)]
    [(Some None)     None]
    [None            None]))
```

### Pattern Guards (:when)

Use `:when` to add a condition to a pattern match.

```lisp
(deftype Value (Num i32) (Zero))

(match val
  [((Num n) :when (> n 100)) "large"]
  [((Num n) :when (> n 0))   "positive"]
  [_                          "non-positive"])
```

An arm is selected only when its pattern matches and its `:when` condition is true. If the guard is `false`, matching continues with the next arm.

### ref / ref mut Patterns

Using `(ref name)` or `(ref mut name)` within a pattern binds a reference without moving the value.

| Syntax | Binding type | Meaning |
|------|--------|------|
| `(ref name)` | `[& T]` | Shared reference — borrows the payload without copying |
| `(ref mut name)` | `[& mut T]` | Mutable reference — mutably borrows the payload |

```lisp
; Shared-reference binding: text has type [& String]
(match event
  [(Message (ref text)) (println! (* text))]
  [Other                ()])

; Mutable-reference binding
(deftype Counter (Count i32))
(defn increment [(c : [& mut Counter])] -> ()
  (match c
    [(Count (ref mut n)) (set! (* n) (+ (* n) 1))]))
```

Reference bindings are particularly useful when a guard (`:when`) must inspect a non-Copy value without moving it. The value is not moved even when the guard is `false`.

### Exhaustiveness Checking

A `match` over an ADT must cover every variant of the scrutinee type (`E-MATCH-0001`). The compiler lists missing variants in the diagnostic message.

```lisp
; Compile-time error E-MATCH-0001: None is not covered
(match opt
  [(Some x) x])
```

The wildcard `_` covers all remaining cases.

```lisp
(match opt
  [(Some x) x]
  [_ 0])
```

Attempting to match a variant after an unguarded arm has already covered it produces `E-MATCH-0002` for an unreachable pattern.

A `match` over scalar literals is lowered to nested `if` expressions and is not subject to ADT exhaustiveness checking. Without a wildcard, an unmatched path implicitly produces `()`. Place `_` last to keep return types consistent and avoid unintended fallthrough.

### Arity (Field-Count) Checking

If the number of field bindings in a pattern does not match the number of fields in the variant, the result is `E-MATCH-0003`. Unneeded fields can be ignored with the wildcard `_`. Repeating a payload binding name other than `_` within the same variant pattern produces `E-MATCH-0004`. Because `_` creates no binding, it may appear multiple times.

```lisp
(deftype Shape (Circle f32) (Rect f32 f32))

; Compile-time error E-MATCH-0003: Circle has one field but two bindings
(match s
  [(Circle r1 r2) ...])

; OK: ignore an unneeded field with _
(match token
  [(Ident name _) name]
  [_              "other"])

; Compile-time error E-MATCH-0004: duplicate binding for the same payload name
(match pair
  [(Pair value value) ...])
```

### Unsupported Patterns

- **Or patterns** — There is no `[(Pat1 | Pat2) body]` form. Use separate arms.
- **Range patterns** — There is no `[0..=100 body]` form. Use a guard (`:when`) instead.

---

## loop / recur

`loop`/`recur` implements tail-recursive loops without consuming stack space.

### Syntax

```lisp
(loop [(variable1 : type1 initial-value1) (variable2 : type2 initial-value2) ...]
  body)
```

`recur` passes new values to the loop variables and returns to the beginning of `loop`.

### Basic Example: Computing a Sum

```lisp
(loop [(i : i32 0) (sum : i32 0)]
  (if (> i 10)
    sum
    (recur (+ i 1) (+ sum i))))
; => 55
```

### Fibonacci Sequence

```lisp
(defn fibonacci [(n : i32)] -> i32
  (loop [(i : i32 0) (a : i32 0) (b : i32 1)]
    (if (== i n)
      a
      (recur (+ i 1) b (+ a b)))))
```

### Rules

- `recur` may be used only in **tail position** within `loop`.
- The number of arguments passed to `recur` must equal the number of `loop` variables.
- Each `recur` argument must have the same type as its corresponding `loop` variable.
- Using `recur` outside tail position is a compile-time error.

### break / continue

`break` and `continue` may be used within `loop`, `for`, and `while`.

```lisp
(let mut i 0)
(loop []
  (set! i (+ i 1))
  (if (< i 4)
    (continue)
    (break)))
```

- `break` and `continue` take no arguments. Passing an argument produces `E-LOWER-0145` for `break` or `E-LOWER-0146` for `continue`.
- `break` exits the nearest loop and returns `()`. A loop that contains `break` has type `()`.
- `continue` ends the current iteration and proceeds to the next one.
- In `for` and `while`, `continue` internally preserves the next-step update.
- In a raw `loop`, `continue` proceeds to the next iteration with the current bindings.

---

## for Expressions

`for` iterates over an integer range. It executes its body for every integer in the range specified by `(range start end)`.

### Syntax

```lisp
(for [variable (range start end)]
  body)
```

### Basic Example

```lisp
(for [i (range 0 10)]
  (println! "{}" i))
```

### Ranges with a Step

```lisp
(for [i (range 0 100 2)]
  (println! "{}" i))
```

`range` accepts an optional third argument that specifies the step.

### Iterating over Collections

`for` can iterate over a `Vec` directly. It can iterate over another concrete type when `Type/iter-len` and `Type/iter-at` can be resolved statically through the `Iterable` protocol. The standard library does not implement `Iterable` for `Array`, so fixed-size arrays must be traversed by index with `(range 0 N)` and `Array/get`. The binding must always be a two-element form, `[variable collection-expression]` (E-LOWER-0252).

```lisp
(defn sum-vec [(xs : [Vec i32])] -> i32
  (let mut total 0)
  (for [x xs]
    (set! total (+ total x)))
  total)
```

### Return Value

A `for` expression returns `()`. To transform a collection, use an array combinator such as `map`.

---

## when Expressions

`when` executes its body only when its condition is true. It corresponds to an `if` with no else branch and returns the unit value `()`.

### Syntax

```lisp
(when condition
  body)
```

### Basic Example

```lisp
(when (> score 100)
  (println! "High score!"))
```

`when` is suitable for conditionally executing operations with side effects. Use `if` to select a value through branching.

---

## unless Expressions

`unless` is the inverse of `when`: it executes its body only when the condition is false. It returns the unit value `()`.

### Syntax

```lisp
(unless condition
  body)
```

### Basic Example

```lisp
(unless (fs/exists? path)
  (println! "File not found"))
```

`unless` is equivalent to `(when (not cond) body)`.

---

## while Expressions

`while` repeatedly executes its body while its condition is true. It returns the unit value `()`.

### Syntax

```lisp
(while condition
  body...)
```

### Basic Example

```lisp
(let mut i 0)
(while (< i 10)
  (println! "{}" i)
  (set! i (+ i 1)))
```

`while` is internally desugared to `loop`/`recur`. Its body may contain multiple expressions.

---

## cond Expressions

`cond` evaluates multiple conditions in order and returns the expression associated with the first true condition. It expresses multiple branches concisely without nested `if` expressions.

### Syntax

```lisp
(cond
  [condition1 expression1]
  [condition2 expression2]
  [true       default-expression])
```

Each clause is a two-element vector, `[condition expression]`. Using `true` (BoolLit) or `:else` (keyword) as the condition of the final clause makes it the default clause. If there is no default clause and no condition matches, `cond` returns `()`.

### Basic Example

```lisp
(let grade (cond
  [(>= score 90) "A"]
  [(>= score 80) "B"]
  [(>= score 70) "C"]
  [true          "F"]))    ; :else is equivalent

(cond
  [(= grade 5) "excellent"]
  [(= grade 4) "good"]
  [:else       "other"])
```

`cond` is desugared into nested `if` expressions.

---

## Logical Operators

`and`, `or`, and `not` are short-circuiting logical operators that are internally desugared into `if` expressions.

### and (Conjunction)

```lisp
(and expression1 expression2 ...)
```

The expressions are evaluated from left to right. Evaluation returns `false` at the first false expression. If every expression is true, it returns the value of the final expression (`true`).

```lisp
(and (> x 0) (< x 100))    ; x is greater than 0 and less than 100
```

`(and a b)` is equivalent to `(if a b false)`.

### or (Disjunction)

```lisp
(or expression1 expression2 ...)
```

The expressions are evaluated from left to right. Evaluation returns `true` at the first true expression. If every expression is false, it returns the value of the final expression (`false`).

```lisp
(or (== x 0) (== x 1))     ; x is 0 or 1
```

`(or a b)` is equivalent to `(if a true b)`.

### not (Negation)

```lisp
(not expression)
```

Returns `false` if the expression is true, or `true` if it is false.

```lisp
(not (== x 0))              ; x is not 0
```

`(not a)` is equivalent to `(if a false true)`.

---

## Control-Flow Summary

| Construct | Purpose | Returns a value? |
|------|------|-----------|
| `if` | Two-way conditional branch | Yes |
| `do` | Sequential execution | Yes (final expression) |
| `match` | Pattern-matching branch | Yes |
| `loop`/`recur` | Tail-recursive loop | Yes |
| `for` | Range traversal | No (returns `()`) |
| `when` | Conditional execution | No (returns `()`) |
| `unless` | Negated conditional execution | No (returns `()`) |
| `while` | Conditional loop | No (returns `()`) |
| `cond` | Multi-way conditional | Yes |
| `and` | Short-circuiting conjunction | Yes |
| `or` | Short-circuiting disjunction | Yes |
| `not` | Logical negation | Yes |
