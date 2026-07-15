# Error Handling

Singulisp has no exception mechanism. Instead, it uses value-based error handling with the `Result` and `Option` types. The `?` operator provides concise error propagation for `Result`, and `Result` and `Option` values can be processed through pattern matching with `match`.

## The Result Type

`[Result T E]` holds either a success value of type `T` or an error value of type `E`.

```lisp
;; Type definition (built in)
(deftype Result [T E]
  (Ok T)
  (Err E))
```

### Constructing Values

```lisp
;; Construct a success value
(Ok 42)           ;; => [Result i32 E]

;; Construct an error value
(Err "not found") ;; => [Result T String]
```

### Basic Example

```lisp
(defn parse-port [(s : String)] -> [Result i32 String]
  (let n (? (String/parse-i64 s)))
  (let n32 (to-i32 n))
  (if (and (>= n32 1) (<= n32 65535))
    (Ok n32)
    (Err "port number out of range")))
```

## The Option Type

`[Option T]` represents either the presence (`Some`) or absence (`None`) of a value.

```lisp
;; Type definition (built in)
(deftype Option [T]
  (Some T)
  None)
```

### Constructing Values

```lisp
;; A value is present
(Some 42)    ;; => [Option i32]

;; No value is present
None         ;; => [Option T]
```

### Basic Example

```lisp
(defn find-record [(id : i32) (records : [Vec Record])] -> [Option Record]
  (loop [(i : i32 0)]
    (if (>= i (Vec/len (& records)))
      None
      (do
        (let record (Vec/get (& records) i))
        (if (== record.id id)
          (Some record)
          (recur (+ i 1)))))))
```

## The ? Operator

The `?` operator automatically propagates a `Result` error to the caller. If the expression is `Err`, it immediately returns that error value from the current function. If it is `Ok`, the operator extracts its inner value.

`?` may be used only when all of the following conditions hold.

- `expr` must have type `[Result T E]`; otherwise, the diagnostic is `E-IR-0160`.
- The function containing `?` must also return a `Result` with error type `E`, such as `[Result U E]`; otherwise, the diagnostic is `E-IR-0163`.
- The `Err` type of `expr` must match the `Err` type of the containing function; a mismatch also produces `E-IR-0163`.
- `?` may also be used inside an active `with-scope`. When propagating an `Err`, it performs a structured join of the tasks in the scope before returning to the caller.

### Using ? with Result

```lisp
(defn read-config [(path : String)] -> [Result Config String]
  (let content (? (fs/read-to-string path)))
  (let parsed (? (parse-config content)))
  (Ok parsed))
```

`?` is not merely a value-producing `match`; its `Err` branch returns early from the current function.

### Chaining Multiple Operations

```lisp
(defn load-state [(path : String)] -> [Result State String]
  (let raw (? (fs/read-to-string path)))
  (let header (? (parse-header raw)))
  (let version (? (validate-version header)))
  (let state (? (deserialize-state raw version)))
  (Ok state))
```

Using `?` at each step immediately returns an `Err` when an error occurs and skips subsequent processing.

## Pattern Matching

Use a `match` expression to branch on a `Result` or `Option` value.

### Matching Result

```lisp
(match result
  [(Ok val) (println! "success: {}" val)]
  [(Err e)  (println! "failure: {}" e)])
```

### Matching Option

```lisp
(match (find-record target-id records)
  [(Some record) (process-record record)]
  [None          (println! "record not found")])
```

### Nested Pattern Matching

```lisp
(match (load-config)
  [(Ok config)
    (if config.debug-mode
      (println! "debug mode enabled")
      (println! "production mode"))]
  [(Err e)
    (println! "failed to load configuration: {}" e)])
```

## assert!

`assert!` is a macro that verifies that a condition is `true`. If the condition is `false`, the program triggers a runtime trap (abort).

```lisp
(assert! (> health 0))
(assert! (< index (Vec/len (& items))))
```

Use `assert!` for debugging and validating invariants. It also runs in production builds, so take care in performance-sensitive code.

## assert-eq!

`assert-eq!` is a macro that verifies that two values are equal. If they differ, the program triggers a runtime trap.

```lisp
(assert-eq! (+ 2 3) 5)
(assert-eq! (Vec/len (& xs)) 10)
```

## assert-ne!

`assert-ne!` is a macro that verifies that two values differ. If they are equal, the program triggers a runtime trap.

```lisp
(assert-ne! divisor 0)
(assert-ne! player-id target-id)
```

## Runtime Traps

The program triggers a runtime trap (immediate abort) in the following situations.

| Trap condition | Description |
|------------|------|
| Out-of-bounds array access | The index is greater than or equal to the array length |
| Division by zero | Integer division by zero |
| OOM (with the `:trap` setting) | A region runs out of memory |
| Failed `assert!` | The condition is `false` |
| Failed `assert-eq!` | The two values differ |
| Failed `assert-ne!` | The two values are equal |

A runtime trap safely stops the program. It does not cause undefined behavior.

### Out-of-Bounds Array Access

```lisp
(let xs (array 1 2 3))
(Array/get xs 10) ;; Runtime trap: index 10 is out of bounds
```

### Division by Zero

```lisp
(let x 42)
(let y 0)
(/ x y) ;; Runtime trap: division by zero
```

## Serialization Error Types

Singulisp provides language-integrated serialization in the BBin v1 binary format. The deserialization return type depends on the entry point.

| Function | Return type |
|------|--------|
| `T/deserialize` | `[Result T String]` |
| `T/try-deserialize` | `[Result T DeserError]` |

An error from `T/deserialize` is a `String` containing a human-readable message. An error from `T/try-deserialize` has type `DeserError`, whose variants are `(Truncated i32)`, `(BadTag i32)`, `BadSchema`, `CycleDetected`, and `(TooLarge i32)`, allowing more precise classification. When using the `?` operator, the containing function's `Err` type must match the selected entry point.

```lisp
;; deserialize: errors are String values
(defn load-data [(bytes : [Vec u8])] -> [Result DataState String]
  (let state (? (DataState/deserialize (& bytes))))
  (Ok state))

;; try-deserialize: errors are DeserError values
(defn safe-load [(bytes : [Vec u8])] -> [Result DataState DeserError]
  (let state (? (DataState/try-deserialize (& bytes))))
  (Ok state))
```

## Practical Error-Handling Patterns

### Providing a Default Value

```lisp
(defn get-or-default [(opt : [Option i32]) (default : i32)] -> i32
  (match opt
    [(Some v) v]
    [None     default]))
```

### Converting an Error

```lisp
(defn load-data [(path : String)] -> [Result Data AppError]
  (match (fs/read-to-string path)
    [(Ok text) (parse-data text)]
    [(Err e)   (Err (AppError/IoError e))]))
```

### Combining Multiple Error Sources

```lisp
#[external]
(defn initialize [] -> [Result Engine String]
  (let config (? (load-config "engine.toml")))
  (let window (? (create-window config.width config.height)))
  (let renderer (? (init-renderer window)))
  (let audio (? (init-audio)))
  (Ok (Engine config window renderer audio)))
```
