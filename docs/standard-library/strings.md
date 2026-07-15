# String Operations

Singulisp provides a byte-string-based string type. Low-level `String/*` built-ins are also available, but normal application code should use `std::strings`.

```lisp
(use std::strings :as str)

(str::starts-with "hello" "he")
(str::parse-i64 "42")  ;; [Option i64]
(format "{}:{}" "x" 10)
```

---

## Types

| Type | Description | Semantics |
|------|-------------|-----------|
| `String` | An owned byte string with SSO, heap, and static-literal representations | Move (ownership transfer) |
| `StrSlice` | A zero-copy borrowed view | Copy (copies a pointer and length) |

```lisp
(let name "Singulisp")           ;; String literal
```

`String` is implemented as a 32-byte struct (conceptually, `{ data, len, cap, hash, flags, reserved }`). Values of 15 bytes or fewer can be stored inline using small-string optimization, longer owned values use the heap, and literals can use static storage. `String/len` returns the byte length as an `i32`, not the number of UTF-8 code points.

`StrSlice` has the same 32-byte physical representation as `String`, but has Copy semantics. **It cannot escape**: returning it or storing it in a struct field, `Vec` element, or `Box` is a compile-time error. Convert it with `(String/from slice)` when an owning `String` is required.

---

## String Function Reference

### `std::strings`

| Function | Signature | Description |
|----------|-----------|-------------|
| `starts-with` | `String String -> bool` | Tests whether a string starts with the specified prefix |
| `ends-with` | `String String -> bool` | Tests whether a string ends with the specified suffix |
| `index-of` | `String String -> i32` | Returns the first byte position, or `-1` if absent |
| `trim` | `String -> String` | Removes leading and trailing ASCII whitespace |
| `replace` | `String String String -> String` | Replaces all occurrences of a substring |
| `split` | `String String -> [Vec String]` | Splits on byte-sequence matches; consecutive separators produce empty elements |
| `chars` | `String -> [Vec i32]` | Returns each byte value as an `i32` |
| `StringBuilder/new` | `() -> StringBuilder` | Creates an empty builder |
| `StringBuilder/with-capacity` | `i32 -> StringBuilder` | Creates a builder with an initial capacity |
| `StringBuilder/append` | `&mut StringBuilder, &String -> ()` | Appends a string |
| `StringBuilder/append-byte` | `&mut StringBuilder, i32 -> ()` | Appends one byte |
| `StringBuilder/append-f64` | `&mut StringBuilder, f64 -> ()` | Appends the representation of an `f64` |
| `StringBuilder/finish` | `&mut StringBuilder -> String` | Extracts the owned string |
| `join` | `&[Vec String], String -> String` | Joins strings with a separator |
| `repeat` | `String i32 -> String` | Repeats a string the specified number of times |
| `pad-left` / `pad-right` | `String i32 i32 -> String` | Pads to a width with a byte value |
| `to-upper` / `to-lower` | `String -> String` | Converts ASCII letter case |
| `f64-to-string` | `f64 -> String` | Converts to Dragonbox shortest round-trip notation |
| `parse-i32` | `String -> [Option i32]` | Returns `Some` on success and `None` on failure or overflow |
| `parse-i64` | `String -> [Option i64]` | Returns `Some` on success and `None` on failure or overflow |
| `parse-f64` | `String -> [Option f64]` | Returns `Some` on success and `None` on failure |

### Layer-1 Built-ins (Primary APIs)

| Function | Signature | Description |
|----------|-----------|-------------|
| `format` | `String args... -> String` | Creates an owned `String` using the same format syntax as `println!` |
| `String/len` | `String -> i32` | Returns the byte length |
| `String/concat` | `String, String -> String` | Concatenates two strings |
| `String/contains` | `String, String -> bool` | Tests whether a string contains a substring |
| `String/starts-with` | `String, String -> bool` | Tests whether a string starts with the specified prefix |
| `String/ends-with` | `String, String -> bool` | Tests whether a string ends with the specified suffix |
| `String/index-of` | `String, String -> i32` | Returns the substring position, or `-1` if not found |
| `String/substring` | `String, i32, i32 -> String` | Extracts a substring using a start position and byte count |
| `String/slice` | `String, i32, i32 -> StrSlice` | Returns a zero-copy view over `[start, end)` |
| `String/from` | `StrSlice -> String` | Materializes a borrowed view as an owned string |
| `String/of-char` | `i32 -> String` | Creates a one-byte string from the low byte of an `i32` |
| `String/char-at` | `String, i32 -> i32` | Returns the byte value at the specified position |
| `String/byte-at` | `String, i32 -> i32` | Explicitly named alias of `String/char-at` |
| `String/to-upper` | `String -> String` | Converts ASCII to uppercase |
| `String/to-lower` | `String -> String` | Converts ASCII to lowercase |
| `String/trim` | `String -> String` | Removes leading and trailing whitespace characters |
| `String/replace` | `String, String, String -> String` | Replaces a substring |
| `String/split` | `String, String -> [Vec String]` | Splits on a separator string |
| `String/f64-to-string` | `f64 -> String` | Converts an `f64` to Dragonbox shortest round-trip notation |
| `String/append-f64-to-vec!` | `&mut [Vec u8], f64 -> ()` | Appends the Dragonbox representation of an `f64` to a `Vec<u8>` |
| `String/eq` | `String, String -> bool` | Compares strings for equality |
| `String/lt` | `String, String -> bool` | Tests whether one string is lexicographically smaller than another |

---

## Function Details and Examples

### `String/len` -- Byte Length

```lisp
(let s "hello")
(let n (String/len s))  ;; n = 5
```

### `String/concat` -- Concatenation

Returns a new string containing the concatenation of two strings.

```lisp
(let greeting (String/concat "hello" " world"))
;; greeting = "hello world"
```

### `String/contains` -- Substring Search

```lisp
(let s "hello world")
(String/contains s "world")  ;; true
(String/contains s "xyz")    ;; false
```

### `String/starts-with` -- Prefix Test

```lisp
(let path "/usr/local/bin")
(String/starts-with path "/usr")  ;; true
```

### `String/ends-with` -- Suffix Test

```lisp
(let file "module.gu")
(String/ends-with file ".gu")  ;; true
```

### `String/index-of` -- Position Search

Returns the zero-based position of the first occurrence of a substring. Returns `-1` when the substring is not found.

```lisp
(let s "hello world")
(String/index-of s "world")  ;; 6
(String/index-of s "xyz")    ;; -1
```

### `String/substring` -- Substring Extraction

Creates an owned string from a start position and length in bytes. The runtime clamps negative values and positions past the end to the valid range.

```lisp
(let s "hello world")
(let sub (String/substring s 0 5))  ;; sub = "hello"
```

### `String/char-at` -- Reading a Byte Value

Returns the value at the specified byte position as an `i32` in the range 0 to 255. `String/byte-at` is an alias of the same function. **No bounds check is performed**; an out-of-range index causes undefined behavior. Selecting a byte in the middle of a multibyte UTF-8 character is not a boundary error: the function returns the byte value at that position, not a code point.

```lisp
(let s "ABC")
(let c (String/byte-at s 0))  ;; c = 65 ('A')
```

### `String/slice` -- Borrowed View

```lisp
(let view (String/slice source start end))
```

Creates a `StrSlice` over `[start, end)` without allocation. Code generation does not check for negative values, invalid ordering, or positions past the end, so the caller must guarantee `0 <= start <= end <= String/len(source)`. Use `String/substring` when a checked and clamped owning copy is required.

### `String/push!` -- Append a Character

```lisp
(String/push! (& mut s) ch)
```

Converts the integer to a `u8` and appends that one byte. It does not encode a Unicode code point as UTF-8. The first argument must be a mutable reference in the form `(& mut <variable>)`.

```lisp
(let mut s "ab")
(String/push! (& mut s) 99)  ;; s is "abc" (99 = 'c')
```

### `format` -- String Formatting

Uses the same `{}` / `{:SPEC}` syntax as `println!` and returns a `String` instead of producing output.

```lisp
(let label (format "{}:{}" "score" 42))
;; label = "score:42"
```

### `std::strings/f64-to-string` -- `f64` Conversion

Converts an `f64` to a string using Dragonbox shortest round-trip notation. The output uses normalized exponential notation: for example, `1.0` becomes `"1E0"`, `1234.0` becomes `"1.234E3"`, and the smallest subnormal becomes `"5E-324"`. Special values are written as `"NaN"`, `"Infinity"`, and `"-Infinity"`.

```lisp
(use std::strings :as str)

(let s (str::f64-to-string 3.141592653589793))
;; s = "3.141592653589793E0"
```

### `String/to-upper` -- Uppercase Conversion

```lisp
(let s "hello")
(let upper (String/to-upper s))  ;; upper = "HELLO"
```

### `String/to-lower` -- Lowercase Conversion

```lisp
(let s "HELLO")
(let lower (String/to-lower s))  ;; lower = "hello"
```

### `String/trim` -- Removing Leading and Trailing Whitespace

```lisp
(let s "  hello  ")
(let trimmed (String/trim s))  ;; trimmed = "hello"
```

### `String/replace` -- Replacement

Replaces every occurrence of the substring.

```lisp
(let s "hello world")
(let r (String/replace s "world" "singulisp"))
;; r = "hello singulisp"
```

### `String/split` -- Splitting

Splits on a separator string and returns the result as `[Vec String]`.

```lisp
(let csv "a,b,c,d")
(let parts (String/split csv ","))
;; parts = ["a" "b" "c" "d"]
```

### `String/eq` -- Equality Comparison

Tests whether two strings are equal.

```lisp
(let a "hello")
(let b "hello")
(String/eq a b)  ;; true
(String/eq a "world")  ;; false
```

### `String/lt` -- Lexicographic Comparison

Tests whether the first argument is lexicographically smaller than the second.

```lisp
(String/lt "abc" "def")  ;; true
(String/lt "def" "abc")  ;; false
```

---

## Parsing Functions

These functions convert strings to numbers.

`std::strings/parse-i64` and `std::strings/parse-f64` return `[Option _]`. Use these functions in normal code that treats failure as a recoverable value.

### `String/parse-i64` -- Integer Parsing

Returns `[Result i64 String]`. Parsing failure produces an `Err`.

```lisp
;; Handle the result with pattern matching
(match (String/parse-i64 "42")
  [(Ok n) (println! "Parse succeeded: {}" n)]
  [(Err msg) (println! "Parse failed: {}" msg)])

;; Propagate errors with the ? operator when the caller returns Result
(let n (? (String/parse-i64 "42")))
```

### `String/parse-f64` -- Floating-Point Parsing

Returns `[Result f64 String]`. Parsing failure produces an `Err`.

```lisp
;; Handle the result with pattern matching
(match (String/parse-f64 "3.14")
  [(Ok x) (println! "Parse succeeded: {}" x)]
  [(Err msg) (println! "Parse failed: {}" msg)])

;; Propagate errors with the ? operator
(let x (? (String/parse-f64 "3.14")))
```

---

## Practical Example: CSV Parser

```lisp
(defn parse-csv-line [(line : String)] -> [Vec String]
  (String/split line ","))

(defn parse-score [(s : String)] -> [Result i64 String]
  (let trimmed (String/trim s))
  (String/parse-i64 trimmed))
```

## Practical Example: Path Operations

```lisp
(defn get-extension [(path : String)] -> String
  (let dot-pos (String/index-of path "."))
  (if (= dot-pos (- 0 1))
    ""
    (String/substring path (+ dot-pos 1) (- (String/len path) (+ dot-pos 1)))))

(defn replace-extension [(path : String) (ext : String)] -> String
  (let dot-pos (String/index-of path "."))
  (let base (if (= dot-pos (- 0 1))
    path
    (String/substring path 0 dot-pos)))
  (String/concat base (String/concat "." ext)))
```
