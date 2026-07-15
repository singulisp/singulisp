# Functions

Functions are the basic building blocks of a Singulisp program. Use `defn` to define named functions and `fn` to create anonymous functions (lambdas).

---

## `defn` syntax

The basic form of a function definition is as follows.

```lisp
(defn function-name [parameter ...]
  body)

(defn function-name [parameter ...] -> return-type
  body)
```

A `parameter` may take either of the following two forms.

```lisp
parameter-name
(parameter-name : type)
```

### Basic examples

```lisp
(defn add [a b] -> i32
  (+ a b))

(defn add [(a : i32) (b : i32)] -> i32
  (+ a b))

#[external]
(defn greet [(name : String)] -> ()
  (println! "Hello, {}!" name))
```

### Multiple expressions

When the body contains multiple expressions, the value of the final expression is the function's return value. No explicit `do` block is required; the body of `defn` is implicitly evaluated in sequence.

```lisp
(defn process [(x : i32)] -> i32
  (let doubled (* x 2))
  (let result (+ doubled 1))
  result)
```

---

## Type annotations

### Parameter types

Parameter type annotations may be omitted. When omitted, the type is determined by HM type inference.

```lisp
(defn square [x] -> i32
  (* x x))

(defn square [(x : i32)] -> i32
  (* x x))
```

### Return type

The return type may also be omitted. Use `-> type` where an explicit type is desirable, such as in a public API.

```lisp
(defn double [x] -> i32
  (* x 2))

(defn double-explicit [(x : i32)] -> i32
  (* x 2))
```

An explicit annotation identical to the inferred result may produce a redundant-annotation warning (`W-TYPE-2001` / `W-TYPE-2002` / `W-TYPE-2003`).

---

## Generic functions

`defn` may include a generic parameter vector immediately after the function name. Singulisp uses distinct syntax for type parameters and const parameters.

```lisp
(defn map-array [T U (N : usize)]
  [(xs : [Array T N]) (f : [Fn [T] -> U])] -> [Array U N]
  ...)
```

### Type parameters

Write a type parameter as a bare symbol.

```lisp
(defn id [T] [(x : T)] -> T
  x)
```

### Const parameters

Declare a compile-time integer, such as the length of a fixed-size collection, in the form `(N : usize)`. `N` is not a runtime variable; it is resolved to a concrete integer during monomorphization.

```lisp
(defn sum-array [(N : usize)] [(xs : [& [Array i32 N]])] -> i32
  (loop [(i : i32 0) (acc : i32 0)]
    (if (>= i (to-i32 N))
      acc
      (recur (+ i 1) (+ acc (Array/get (* xs) i))))))
```

### Constraints

- Const parameters support only `usize`.
- `_` does not trigger implicit size inference.
- No unresolved const expressions remain in IR or MIR.

---

## Namespaced form

Built-in and standard-library functions are called by names qualified with a namespace, separated by a slash (`/`).

```lisp
; Mathematical functions
(f32/sqrt 2.0f)
(f64/abs -3.14)
(f32/sin 1.57f)
(f32/cos 0.0f)

; String operations
(String/len s)
(String/concat a b)
(String/contains s "hello")

; Vec operations
(Vec/push (& mut v) elem)
(Vec/len (& v))
(Vec/get (& v) 0)

; File I/O
(fs/read-to-string "path.txt")
(fs/write-string "out.txt" content)
(fs/read-range "asset.bin" 128 64)
(stream/open-reader "asset.bin")

; Utilities
(io/read-line)
(time/now-ms)
(process/exit 0)
```

As shorthand, some functions such as `sqrt` and `abs` may also be called without a prefix.

```lisp
(sqrt 2.0f)     ; equivalent to f32/sqrt
(abs -3.14f)    ; equivalent to f32/abs
```

---

## Lambdas (anonymous functions)

Use `fn` to create an anonymous function.

```lisp
(fn [x] -> i32 (* x 2))
```

It may also be bound to a variable and then used.

```lisp
(let double (fn [x] (* x 2)))
(double 5)  ; => 10
```

Because the head of a call form must be a symbol, the `((fn ...) value)` form, with a lambda expression directly in the head position, is not allowed. To call a lambda, first bind it to a local name as in the example above.

---

## Closures

A lambda expression may capture variables from the scope in which it is defined. Such a lambda is called a closure.

```lisp
(let offset 10)
(let add-offset (fn [(x : i32)] -> i32 (+ x offset)))
(add-offset 5)  ; => 15
```

### Capture and ownership

Captured variables follow the ownership rules.

- Capturing a Copy value copies it into the closure environment.
- Capturing a Move value transfers ownership into the closure environment.
- An ordinary escaping closure cannot capture borrowed references or region handles.
- Non-escaping exception: a closure passed directly to certain `Array` combinators may capture shared and mutable borrows.
- Task closure: a fresh closure passed directly to `scope/spawn` or `sync` may capture async-safe region handles and owned values when it satisfies the region policy and join-before-exit checks.

For details of task boundaries, see [Structured Concurrency](concurrency.md).

```lisp
(let name "Singulisp")
(let greet (fn [] -> () (println! "Hello, {}!" name)))
; name is captured by the closure
(greet)
```

---

## Higher-order functions

A function that takes a function as an argument or returns one as its result is called a higher-order function.

### Taking a function as an argument

```lisp
(defn apply-twice [(f : [Fn [i32] -> i32]) (x : i32)] -> i32
  (f (f x)))

(defn increment [(x : i32)] -> i32
  (+ x 1))

(apply-twice increment 5)  ; => 7
```

### Passing a lambda

```lisp
(apply-twice (fn [(x : i32)] -> i32 (* x 3)) 2)  ; => 18
```

### Array combinators

Higher-order functions are particularly useful for array operations. Singulisp provides the following array combinators.

```lisp
; Array/map: apply a function to each element
(Array/map arr (fn [(x : i32)] -> i32 (* x 2)))

; Array/fold: fold elements
(Array/fold 0 arr (fn [(acc : i32) (x : i32)] -> i32 (+ acc x)))

; Array/filter-map!: combine filtering and mapping (mutating)
(Array/filter-map! arr
  (fn [(x : i32)] -> bool (> x 0))
  (fn [(x : i32)] -> i32 (* x 2)))
```

---

## Methods

Using `defn` inside an `impl` block defines a method associated with a type.

### The `self` parameter

When `self` is the first parameter of a method, the method is an instance method of that type.

```lisp
(defstruct Counter
  (value : i32))

(impl Counter
  (defn new [] -> Counter
    (Counter 0))

  (defn get [(self : Counter)] -> i32
    self.value)

  (defn increment [(self : Counter)] -> Counter
    (Counter (+ self.value 1))))
```

### Method calls

Call a method in the form `ConcreteType/method`. A function defined within an `impl` block is automatically prefixed with the type name.

```lisp
(let c (Counter/new))
(let c2 (Counter/increment c))
(println! "{}" (Counter/get c2))  ; => 1
```

---

## Recursion

### Direct recursion

A function may recurse by calling itself directly.

```lisp
(defn factorial [(n : i32)] -> i32
  (if (<= n 1)
    1
    (* n (factorial (- n 1)))))
```

### Tail-call optimization with `loop`/`recur`

Using `loop`/`recur` transforms recursion in tail position into a loop that does not consume stack space. This form is recommended when deep recursion is required.

```lisp
(defn factorial [(n : i32)] -> i32
  (loop [(i : i32 n) (acc : i32 1)]
    (if (<= i 1)
      acc
      (recur (- i 1) (* acc i)))))
```

`recur` must occur in the tail position of a `loop`. Using it anywhere else is a compile-time error.

---

## The `#[external]` attribute

Mark a function that invokes external side effects, such as I/O or FFI, with the `#[external]` attribute.

```lisp
#[external]
(defn write-file [(path : String) (content : String)] -> ()
  (fs/write-string path content))
```

Calling an external or diagnostics function directly from an ordinary unannotated function is a compile-time error. Ordinary functions may still perform local mutation, allocation, and traps, so this does not imply complete effect inference or referential transparency. For the rules, see [Effect Call Boundaries](effects.md).

---

## `const-fn` (compile-time functions)

A function defined with the `const-fn` keyword can be evaluated at compile time.

```lisp
(const-fn max-size [] -> i32 1024)

;; Use the result of const-fn as a compile-time constant
(const BUF_SIZE : i32 (max-size))
```

The body of a const-fn may use only expressions that can be evaluated at compile time. I/O, heap allocation, and similar operations are not permitted. Supported operations are `+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `<=`, `>`, `>=`, `if`, recursive calls, `(array ...)`, and `(Array/fill ...)`.

---

## `extern` (foreign function declarations)

Declare foreign functions, such as C functions, with `extern` before calling them.

```lisp
(extern "C"
  (defn puts [(s : String)] -> i32)
  (defn malloc [(size : u64)] -> u64)
  (defn free [(ptr : u64)] -> ()))
```

A foreign function call is implicitly treated as `#[external]`. At the surface level, pointers are represented primarily as `u64`. For details, including String termination and length contracts, see [C FFI](../platform/ffi.md).
