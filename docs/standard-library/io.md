# Input and Output

Because Singulisp's input/output functions have side effects, their caller must have the `#[external]` or `#[diagnostics]` attribute. The type system prevents accidental calls to I/O functions from pure functions.

---

## Standard Output

### `println!` -- Formatted Output

```lisp
(println! "format string" args...)
```

Takes a format string and arguments, then writes them to standard output followed by a newline. Arguments fill `{}` placeholders in order.

The calling function must have the `#[external]` or `#[diagnostics]` attribute. Using `println!` inside a pure function produces error `E-EFF-0001`. Unlike `format`, `println!` cannot be used without one of these attributes.

#### Supported Types

| Type | Output Format |
|------|---------------|
| `i8`, `i16` | Decimal integer |
| `i32` | Decimal integer |
| `i64` | Decimal integer |
| `u8`, `u16` | Unsigned decimal integer |
| `u32` | Unsigned decimal integer |
| `u64` | Unsigned decimal integer |
| `f32` | Floating-point number |
| `f64` | Floating-point number |
| `bool` | `true` or `false` |
| `String` | The string itself |
| `StrSlice` | The string in the borrowed range |

#### Examples

```lisp
;; Basic output
(println! "hello world")

;; Interpolating values
(let x 42)
(let y 3.14)
(println! "x = {}, y = {}" x y)

;; Interpolating a string
(let name "Singulisp")
(println! "Language: {}" name)

;; Printing a bool
(let flag true)
(println! "Enabled: {}" flag)  ;; Enabled: true

;; Multiple values
(let a 1)
(let b 2)
(let c (+ a b))
(println! "{} + {} = {}" a b c)  ;; 1 + 2 = 3
```

---

## Standard Input

### `io/read-line` -- Read One Line

```lisp
(io/read-line)
```

Reads one line from standard input and returns it as a `String`. The trailing newline is not included.

```lisp
#[external]
(defn ask-name [] -> String
  (println! "Enter your name:")
  (io/read-line))
```

---

## File and Stream I/O

Most file-operation functions return a `Result`. On error, the error message is stored as a `String`. Errors can be propagated with the `?` operator.

These are side-effecting APIs, so the caller must have `#[external]`. A `defbench` body is a pure context, so it cannot call `fs/*` or `stream/*` directly.

### `std::io`

Normal code should use the Result-based API in `std::io`. The `fs/*` functions are low-level built-ins; `std::io` returns their errors wrapped in the `IoError` enum.

```lisp
(use std::io)

#[external]
(defn first-byte [(path : [& String])] -> [Result u8 IoError]
  (let bytes (? (File/read-all path)))
  (if (> (Vec/len (& bytes)) 0)
    (Ok (Vec/get (& bytes) 0))
    (Err (Other -1))))
```

| Function | Type | Description |
|----------|------|-------------|
| `File/exists?` | `[& String] -> bool` | Tests whether a path exists |
| `File/read-all` | `[& String] -> [Result [Vec u8] IoError]` | Reads an entire file as bytes |
| `File/write-all` | `[& String] [& [Vec u8]] -> [Result () IoError]` | Writes bytes, truncating any existing contents |

`IoError` has the variants `NotFound`, `Permission`, and `Other i32`. The runtime wraps the `String` errors from the existing `fs/*` API, so detailed error numbers other than `NotFound` are reduced to `Other -1`.

### Whole-File API

| Function | Type | Description |
|----------|------|-------------|
| `fs/read-to-string` | `String -> [Result String String]` | Reads all file bytes into a `String` without UTF-8 validation |
| `fs/write-string` | `String String -> [Result () String]` | Writes a string, truncating any existing contents |
| `fs/exists?` | `String -> bool` | Tests whether a path exists |
| `fs/delete` | `String -> bool` | Deletes a path |
| `fs/size` | `String -> [Result i64 String]` | Returns the file size |
| `fs/read-bytes` | `String -> [Result [Vec u8] String]` | Reads an entire file as bytes |
| `fs/read-range` | `String i64 i64 -> [Result [Vec u8] String]` | Reads only the specified range |
| `fs/write-bytes` | `String [Vec u8] -> [Result () String]` | Writes bytes, truncating any existing contents |
| `fs/append-string` | `String String -> [Result () String]` | Appends a string |
| `fs/append-bytes` | `String [Vec u8] -> [Result () String]` | Appends bytes |

### `fs/read-to-string`

```lisp
(fs/read-to-string path)
```

Reads an entire file as a byte sequence into a `String`. The runtime does not validate UTF-8, so the data has the same byte semantics as the string API.

```lisp
#[external]
(defn load-config [(path : String)] -> [Result String String]
  (fs/read-to-string path))
```

### `fs/write-string`

```lisp
(fs/write-string path content)
```

Opens the file, truncates it, and writes the string.

```lisp
#[external]
(defn save-data [(path : String) (data : String)] -> [Result () String]
  (fs/write-string path data))
```

### `fs/read-range`

```lisp
(fs/read-range path start len)
```

- `start`: Offset at which reading begins
- `len`: Number of bytes to read
- Return value: `[Result [Vec u8] String]`

`fs/read-range` uses seek and read to process only the requested range. It does not read the entire file and then extract the range.

```lisp
#[external]
(defn read-header [(path : String)] -> [Result [Vec u8] String]
  (fs/read-range path 0 64))
```

### Append APIs

```lisp
(fs/append-string path content)
(fs/append-bytes path bytes)
```

Both append to the end while preserving the existing contents.

### `fs/exists?` / `fs/delete`

```lisp
(fs/exists? path)
(fs/delete path)
```

- `fs/exists?`: Returns `true` when the path exists
- `fs/delete`: Returns `true` on successful deletion and `false` on failure

```lisp
#[external]
(defn ensure-file [(path : String)] -> ()
  (if (not (fs/exists? path))
    (let _ (fs/write-string path ""))
    ()))
```

### Stream API

Use `stream/*` for long-lived sequential I/O. `Reader` and `Writer` are move-only opaque values that wrap host OS handles. Call `stream/close` explicitly when they are no longer needed.

| Function | Type | Description |
|----------|------|-------------|
| `stream/open-reader` | `String -> [Result Reader String]` | Opens a stream for reading |
| `stream/open-writer` | `String i32 -> [Result Writer String]` | Opens a stream for writing |
| `stream/read` | `[& mut Reader] i64 -> [Result [Vec u8] String]` | Reads the specified number of bytes |
| `stream/read-into` | `[& mut Reader] [& mut [Vec u8]] -> [Result i64 String]` | Reads directly into an existing buffer |
| `stream/read-line` | `[& mut Reader] -> [Result String String]` | Reads one line |
| `stream/write` | `[& mut Writer] [& [Vec u8]] -> [Result i64 String]` | Writes bytes |
| `stream/flush` | `[& mut Writer] -> [Result () String]` | Flushes the writer |
| `stream/seek` | `[& mut Reader/Writer] i64 i32 -> [Result i64 String]` | Moves the offset |
| `stream/close` | `[& mut Reader/Writer] -> [Result () String]` | Closes the stream |

#### Mode and Whence

- `(stream/mode-truncate)`: Create and truncate
- `(stream/mode-append)`: Append
- `(stream/whence-set)`: `SEEK_SET`
- `(stream/whence-cur)`: `SEEK_CUR`
- `(stream/whence-end)`: `SEEK_END`

Pass these public constant functions as the final argument to `stream/open-writer` or `stream/seek`.

#### Example

```lisp
#[external]
(defn copy-prefix [(src : String) (dst : String)] -> [Result () String]
  (match (stream/open-reader src)
    [(Err e) (Err e)]
    [(Ok reader0)
      (do
        (let mut reader reader0)
        (let chunk (? (stream/read (& mut reader) 128)))
        (? (stream/close (& mut reader)))
        (match (stream/open-writer dst (stream/mode-truncate))
          [(Err e) (Err e)]
          [(Ok writer0)
            (do
              (let mut writer writer0)
              (? (stream/write (& mut writer) (& chunk)))
              (? (stream/flush (& mut writer)))
              (? (stream/close (& mut writer)))
              (Ok ()))]))]))
```

---

## Error Propagation with the `?` Operator

Because file I/O functions return `Result`, the `?` operator can propagate errors concisely. On an `Err`, `?` immediately returns it to the caller.

```lisp
#[external]
(defn process-file [(path : String)] -> [Result String String]
  (let content (? (fs/read-to-string path)))
  (let processed (String/to-upper content))
  (? (fs/write-string "output.txt" processed))
  (Ok processed))
```

---

## The `#[external]` Attribute

Functions containing I/O operations must be marked with the `#[external]` or `#[diagnostics]` attribute. These attributes allow call boundaries to be checked statically; they are not a general-purpose effect system or a complete proof that a function body is pure.

```lisp
;; Correct: has the external attribute
#[external]
(defn main [] -> ()
  (println! "hello"))

;; Correct: the diagnostics attribute is also accepted
#[diagnostics]
(defn log-diagnostic [(msg : String)] -> ()
  (println! "Diagnostic: {}" msg))

;; Compile-time error: no attribute
(defn bad [] -> ()
  (println! "hello"))  ;; E-EFF-0001: a pure function cannot call a side-effecting function
```

See [external / diagnostics](../language/effects.md) for the detailed boundary rules.

---

## Practical Example: Writing a Log File

```lisp
#[external]
(defn write-log [(message : String)] -> ()
  (let line (String/concat "[LOG] " message))
  (let _ (fs/write-string "app.log" line))
  ())

#[external]
(defn main [] -> ()
  (write-log "Processing started")
  (println! "Processing has started"))
```

## Practical Example: Reading a Configuration File

```lisp
#[external]
(defn load-settings [(path : String)] -> [Result String String]
  (if (fs/exists? path)
    (fs/read-to-string path)
    (Err "Configuration file not found")))
```
