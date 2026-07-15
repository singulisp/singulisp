# Option / Result Combinators

`Option` and `Result` are built-in ADTs (algebraic data types) predefined by the
compiler. Users cannot redefine them. The variant names `Some`, `None`, `Ok`, and
`Err` are reserved as well.

`std::option_result` provides layer-2 `Option/*` and `Result/*` combinators.

```lisp
(use std::option_result)

(Option/map (Some 21) (fn [(x : i32)] -> i32 (* x 2)))
(Result/and-then (Ok 3) (fn [(x : i32)] -> [Result i32 String] (Ok (* x 4))))
```

| Function | Type |
|------|----|
| `Option/map` | `[Option T] [Fn [T] -> U] -> [Option U]` |
| `Option/and-then` | `[Option T] [Fn [T] -> [Option U]] -> [Option U]` |
| `Option/or` | `[Option T] [Option T] -> [Option T]` |
| `Option/ok-or` | `[Option T] E -> [Result T E]` |
| `Option/expect` | `[Option T] String -> T` |
| `Result/map` | `[Result T E] [Fn [T] -> U] -> [Result U E]` |
| `Result/and-then` | `[Result T E] [Fn [T] -> [Result U E]] -> [Result U E]` |
| `Result/or` | `[Result T E] [Result T E] -> [Result T E]` |
| `Result/expect` | `[Result T E] String -> T` |

`expect` traps on failure and reports the supplied `message` as a diagnostic.

---

## The `?` Operator (Error Propagation)

```lisp
(? expr)
```

`expr` must have type `[Result T E]`. For `(Ok v)`, the operator extracts `v` in
place. For `(Err e)`, it returns the same `Err` from the calling function. `?` cannot
be applied to `Option`.

Requirements:

- The calling function must return `[Result T E]`.
- For `(Err e)`, error type `E` must match the error type in the calling function's
  `Result`.

```lisp
#[external]
(defn load-and-parse [(path : String)] -> [Result i64 String]
  (let content (? (fs/read-to-string path)))
  (? (String/parse-i64 (String/trim content))))
```

---

## The `DeserError` Type

`DeserError` is a built-in enum predefined by the compiler and used as the error type
for `T/try-deserialize`. Users do not need to define it.

| Variant | Payload | Description |
|-----------|-----------|------|
| `Truncated` | `i32` | The byte sequence ends prematurely |
| `BadTag` | `i32` | An enum tag is out of range |
| `BadSchema` | None | The magic or version is invalid, or the header size is inconsistent despite an identical schema |
| `CycleDetected` | None | The recursion-depth limit of 256 was exceeded, or an invalid migration descriptor was detected |
| `TooLarge` | `i32` | A length field exceeds the permitted limit |

The `i32` payload supplies auxiliary information for locating the error. It does not
always represent a byte count or the received tag.

```lisp
(match (Player/try-deserialize (& bytes))
  [(Ok p)             (println! "Load succeeded")]
  [(Err (Truncated _)) (println! "Incomplete data")]
  [(Err (BadSchema))   (println! "Invalid serialization header")]
  [(Err _)             (println! "Deserialization error")])
```
