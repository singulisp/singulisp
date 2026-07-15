# Syntax

Singulisp source code uses the S-expression as its basic unit. This chapter comprehensively defines the language's lexical structure and syntactic elements, from source-text encoding rules through literals, identifiers, and special forms.

---

## Source Text

Singulisp source files must be encoded in **UTF-8**. The compiler automatically removes a BOM (U+FEFF) at the beginning of a file.

Newline normalization:

- `\r\n` (CR+LF) is converted to `\n` (LF).
- A bare `\r` (CR, U+000D) must not appear in source.
- A NUL character (U+0000) must not appear in source.

The compiler reports an error if any of these prohibited characters is detected.

---

## Comments

### Line Comments

Text from a semicolon (`;`) to the end of the line is a comment. A semicolon inside a string literal does not begin a comment.

```lisp
; This is a line comment
(defn add [(a : i32) (b : i32)] -> i32
  (+ a b))  ; inline comment
```

### Block Comments

Block comments of the form `#| ... |#` are **not supported** (E-LEX-0031). For a comment spanning multiple lines, prefix each line with `;`.

```lisp
; A multiline comment places
; a semicolon on each line like this
```

### Documentation Comments

A line beginning with `;;;` (three semicolons) is a documentation comment. The compiler retains it as a distinct token and stores it in the documentation field of the immediately following form. If exactly one space follows `;;;`, that space is removed; if several spaces follow, only one is removed.

```lisp
;;; Returns the sum of two integers.
;;; Also works correctly with negative arguments.
(defn add [(a : i32) (b : i32)] -> i32
  (+ a b))
```

---

## Delimiters

The following characters delimit tokens and form syntactic elements.

| Character | Purpose |
|------|------|
| `(` `)` | List (the basic S-expression structure) |
| `[` `]` | Vector (type parameters, parameter lists, and so on) |
| `{` `}` | Reserved and unsupported (`E-READ-0023` when used alone) |
| `"` | Beginning and end of a string literal |

Whitespace (spaces, tabs, and newlines) separates tokens.

---

## Integer Literals

Integer literals support the following radix notations.

### Decimal

```lisp
42
0
1000000
```

### Hexadecimal

Uses a `0x` or `0X` prefix and the digits `0-9`, `a-f`, and `A-F`.

```lisp
0xFF
0x1A2B
```

### Binary

Uses a `0b` or `0B` prefix and the digits `0` and `1`.

```lisp
0b1010
0b11111111
```

### Signed Literals

When an ASCII digit immediately follows `+` or `-`, the sequence is parsed as a signed integer literal. When separated by a space, the operator and integer are separate tokens.

```lisp
-128    ; IntLit(-128)
+10     ; IntLit(10)
- 3     ; operator "-" and IntLit(3) (separate tokens)
```

Integer literals do not support type suffixes. Width is inferred from context. If context does not determine a width, the default depends on the value: `i32` when the value fits in the `i32` range, otherwise `i64` or `u64`. Integer literals have no digit separator such as `1_000_000`; placing `_` within a number is an error.

---

## Floating-point Literals

A floating-point literal is a number containing a decimal point or exponent part.

```lisp
3.14        ; f64
3.14f       ; f32 (f suffix)
1.0e-3      ; exponential notation (1.0 * 10^-3 = 0.001)
2.5e10      ; exponential notation (2.5 * 10^10)
```

| Suffix | Type |
|--------|----|
| `f` | f32 |
| none | f64 |

---

## String Literals

A string literal is enclosed in double quotes (`"`).

```lisp
"hello"
"Unicode: café"
""              ; empty string
```

### Escape Sequences

| Sequence | Meaning |
|-----------|------|
| `\n` | Newline (LF) |
| `\t` | Tab |
| `\r` | Carriage return (CR) |
| `\\` | Backslash |
| `\"` | Double quote |
| `\u{HHHH}` | Unicode code point (one to six hexadecimal digits) |

```lisp
"line1\nline2"
"tab\tseparated"
"Unicode: \u{03B1}"  ; α
```

---

## Character Literals

A character literal consists of the `#\` prefix followed by a character or named character. It may be used in expression position and desugars to an integer literal holding the Unicode code-point value.

### Single Characters

```lisp
#\a         ; 'a'
#\Z         ; 'Z'
#\0         ; '0'
```

### Named Characters

| Literal | Character |
|---------|------|
| `#\newline` | Newline (LF, U+000A) |
| `#\tab` | Tab (U+0009) |
| `#\space` | Space (U+0020) |

---

## Keywords

A keyword is a symbol beginning with a colon (`:`), used primarily for named arguments and option specifications.

```lisp
:key
:on-oom
:bump
:repr
:width
:hint
:fast
:contract
:erase-call
:stub
:error
:on-missing
```

A keyword is not evaluated as a value; it acts as a syntactic modifier.

---

## Booleans

Boolean values are represented by the two literals `true` and `false`.

```lisp
true
false
```

Their type is `bool`.

---

## Identifiers

Identifiers are limited to ASCII characters and follow these rules.

### Basic Rules

- First character: `[A-Za-z_]`
- Subsequent characters: `[A-Za-z0-9_!?-]` (including hyphen `-`, exclamation mark `!`, and question mark `?`)

```lisp
x
my-value
empty?
println!
_unused
Vec3
```

### Namespaced Form

A slash (`/`) separates the namespace in a qualified identifier.

```lisp
f32/sqrt           ; sqrt in the f32 module
String/len         ; string length
Vec/push           ; append to a vector
io/read-line       ; read input
fs/read-to-string  ; read a file
```

### Field Access

A dot (`.`) denotes field access.

```lisp
point.x
self.name
transform.position
```

---

## Region Names

A region name has an at sign (`@`) as its prefix. It specifies the destination region for memory allocation.

```lisp
@frame          ; frame region
@temp           ; temporary region
```

---

## Lifetime Names

A lifetime name has a caret (`^`) as its prefix. It annotates the lifetime of a reference.

```lisp
^a              ; lifetime a
^lifetime       ; lifetime named lifetime
```

```lisp
(defn first [(items : [& ^a [Vec i32]])] -> i32
  (Vec/get items 0))
```

---

## Operators

Singulisp uses operators in prefix notation. The following binary operators are built in.

### Arithmetic Operators

| Operator | Meaning |
|--------|------|
| `+` | Addition |
| `-` | Subtraction |
| `*` | Multiplication |
| `/` | Division |
| `%` | Remainder (`rem` is synonymous) |

```lisp
(+ 1 2)        ; => 3
(* 3 (+ 4 5))  ; => 27
(% 10 3)       ; => 1
```

### Comparison Operators

| Operator | Meaning |
|--------|------|
| `<` | Less than |
| `>` | Greater than |
| `<=` | Less than or equal |
| `>=` | Greater than or equal |
| `==` | Equal (`=` is synonymous) |
| `!=` | Not equal |

```lisp
(< 1 2)        ; => true
(== x y)        ; whether x and y are equal
```

### Bitwise Operators

| Operator | Meaning |
|--------|------|
| `bit-and` | Bitwise AND |
| `bit-or` | Bitwise OR |
| `bit-xor` | Bitwise XOR |
| `bit-not` | Bitwise NOT |
| `bit-shl` | Left shift |
| `bit-shr` | Right shift |

```lisp
(bit-and 0xFF 0x0F)  ; => 0x0F
(bit-shl 1 4)        ; => 16
```

---

## Hash-prefixed Syntax

Only the following two kinds of token beginning with `#` are accepted.

| Syntax | Purpose |
|------|------|
| `#[...]` | Attribute |
| `#\character` | Character literal |

A bare `{` produces `E-READ-0023`. Use `HashMap` or `PMap` from `std::collections` for a runtime map.

---

## S-expression Reader

The Singulisp reader desugars the following prefixes into CST forms. In practice, macro templates use backquote, unquote, and unquote-splicing.

| Syntax | Expansion | Purpose |
|------|------|------|
| `'x` | `(quote x)` | Reader desugaring only; no runtime evaluator for simple quote is provided |
| `` `x `` | `(quasiquote x)` | Quasiquote—partially evaluates within a template |
| `~x` | `(unquote x)` | Unquote—evaluates an expression within quasiquote |
| `~@xs` | `(unquote-splicing xs)` | Splicing—expands and inserts a list |

```lisp
(defmacro unless [cond body]
  `(if ~cond () ~body))
```

`'x` and `(quote x)` do not create runtime data values and therefore cannot be used in ordinary code.

---

## Special Forms

The following are Singulisp's special forms. Unlike macros and functions, these are syntactic elements interpreted directly by the compiler.

### Definitions

| Form | Purpose |
|------|------|
| `defn` | Function definition |
| `defstruct` | Struct definition |
| `deftype` | Algebraic data type (enum) definition |
| `deftrait` | Trait definition |
| `defregion` | Region definition |
| `defmacro` | Macro definition |
| `deftest` | Test definition |
| `defbench` | Benchmark definition |
| `deftype-fn` | Type-function definition (determines representation type from type parameters) |
| `impl` | Trait or method implementation |

### Binding and Assignment

| Form | Purpose |
|------|------|
| `let` | Immutable local variable binding; `(let mut name value)` declares a mutable variable |
| `const` | Compile-time constant |
| `set!` | Reassignment of a variable |

### Control Flow

| Form | Purpose |
|------|------|
| `if` | Conditional branch |
| `do` | Sequential execution block |
| `match` | Pattern matching |
| `loop` | Loop, combined with `recur` |
| `recur` | Tail-recursive call |
| `for` | Range traversal |
| `when` | Conditional execution without an else branch |
| `while` | Conditional loop |
| `unless` | Executes when a condition is false |
| `cond` | Multi-branch conditional expression |
| `and` | Logical conjunction with short-circuit evaluation |
| `or` | Logical disjunction with short-circuit evaluation |
| `not` | Logical negation |
| `?` | Try-unwrap (`Result` error propagation) |

### Reference Operations

| Form | Purpose |
|------|------|
| `&` | Creates a reference: `(& expr)` for a shared reference, `(& mut expr)` for a mutable reference |
| `*` | Dereferences a reference with `(* expr)` |

### Memory Management

| Form | Purpose |
|------|------|
| `@region-name` | Region block, for example `(@frame fa body...)` |
| `arena/alloc!` | Allocates a value in an arena: `(arena/alloc! handle value)` |
| `arena/try-alloc` | Fallible arena allocation; returns `Option` or `Result` according to region policy |
| `persist` | Persists data between regions: `(persist @region value)` |
| `thaw` | Restores persistent data: `(thaw @region alloc resolver value)` |
| `freeze` | Freezes data: `(freeze value)` |

### Concurrency

| Form | Purpose |
|------|------|
| `with-scope` | Creates a structured-concurrency scope: `(with-scope s body...)` |
| `scope/spawn` | Spawns a task within a scope: `(scope/spawn s expr)` |
| `sync` | Executes tasks synchronously: `(sync fn1 fn2 ...)` |
| `channel/new` | Creates a channel: `(channel/new buffer-size)` |
| `channel/send` | Sends a message to a channel: `(channel/send ch value)` |
| `channel/recv` | Receives a message from a channel: `(channel/recv ch)` |
| `channel/close` | Closes a channel: `(channel/close ch)` |
| `spmd` | SPMD parallel-execution block: `(spmd [i N] body...)` |

### GPU

| Form | Purpose |
|------|------|
| `gpu/dispatch` | Dispatches a GPU kernel: `(gpu/dispatch shader :threads [x y z] :args [...])` |

### Compile Time

| Form | Purpose |
|------|------|
| `const-fn` | Compile-time function definition |
| `const-gen` | Compile-time code generation; return type is `[Expr T]` |
| `expr` | Typed staging expression; `(expr body)` converts to `Expr<T>` |

### Other Forms

| Form | Purpose |
|------|------|
| `fn` | Anonymous function (lambda) |
| `extern` | Foreign function declaration (FFI) |
| `use` | Module import |
| `unsafe` | Explicit syntactic boundary for unsafe operations; does not relax type, ownership, or borrow checking |
