# Documentation Generation

Singulisp can generate API documentation automatically from source code. The `gu-cli doc` command combines documentation comments and type information into a Markdown reference.

---

## Basic Usage

```bash
# Generate in the default doc/ directory
gu-cli doc src/lib.gu

# Select an output directory
gu-cli doc src/lib.gu -o api-docs/

# Generate for an entire project
gu-cli doc . -o api-docs/

# Write to stdout for MCP integration
gu-cli doc src/lib.gu --output -
```

The input may be a single `.gu` file or a project directory. For a single file, the output file name is the input stem with `.md` appended; for example, `math.gu` produces `doc/math.md`.

---

## Generation Process

1. Read the source, expand macros, and perform type analysis.
2. Collect public definitions marked with `#[pub]`, grouped by kind.
3. Construct Markdown from type information and documentation comments.
4. If the output destination is `-`, write to stdout; otherwise, write to `<output_dir>/<pkg_name>.md`.

---

## Documentation Comments

Documentation comments begin with three semicolons (`;;;`). Normal comments beginning with `;` or `;;` are not included in the documentation.

```lisp
;;; Calculates the length of a vector.
;;;
;;; Returns the Euclidean norm (L2 norm).
;;; Computes the length from the input vector's x, y, and z components.
#[pub]
(defn length [(v : Vec3)] -> f32
  (f32/sqrt (+ (* v.x v.x) (+ (* v.y v.y) (* v.z v.z)))))
```

### Comment Placement

Place a documentation comment immediately before its definition, or before the attributes when the definition has attributes.

```lisp
;;; Calculates the dot product of two vectors.
#[pub]
(defn dot [(a : Vec3) (b : Vec3)] -> f32
  (+ (* a.x b.x) (+ (* a.y b.y) (* a.z b.z))))
```

### Multiline Documentation

Separate paragraphs with an empty documentation-comment line containing only `;;;`.

```lisp
;;; Calculates the cross product of two vectors.
;;;
;;; Returns the cross-product vector in a right-handed coordinate system.
;;; The resulting vector is orthogonal to both input vectors.
;;;
;;; Returns the zero vector when the input vectors are parallel.
#[pub]
(defn cross [(a : Vec3) (b : Vec3)] -> Vec3
  (Vec3
    (- (* a.y b.z) (* a.z b.y))
    (- (* a.z b.x) (* a.x b.z))
    (- (* a.x b.y) (* a.y b.x))))
```

---

## Visibility and Documentation Generation

Only public definitions marked with `#[pub]` are included. Package-internal functions and definitions marked with `#[priv]` are omitted.

```lisp
;;; This function is included in the documentation.
#[pub]
(defn public-api [(x : i32)] -> i32
  (internal-helper x))

;;; This function is not included because it is package-internal.
(defn internal-helper [(x : i32)] -> i32
  (* x 2))
```

---

## Collected Definitions

Only definitions marked with `#[pub]` are collected for documentation generation.

## Documented Definition Kinds

The following definition kinds are documented.

### Functions

```lisp
;;; Finds the greatest common divisor of two integers.
#[pub]
(defn gcd [(a : i32) (b : i32)] -> i32
  (if (== b 0) a (gcd b (% a b))))
```

Generated documentation:

```markdown
### `gcd (a : i32) (b : i32) -> i32`

Finds the greatest common divisor of two integers.
```

### Structs

```lisp
;;; A struct representing a three-component measurement.
;;;
;;; Each component is stored as a single-precision floating-point number.
#[pub]
(defstruct Measurement3 (x : f32) (y : f32) (z : f32))
```

Generated documentation:

```markdown
### `Measurement3`

A struct representing a three-component measurement.

Each component is stored as a single-precision floating-point number.

| Field | Type |
|-------|------|
| x | f32 |
| y | f32 |
| z | f32 |
```

### Enums

```lisp
;;; The state of a processing job.
#[pub]
(deftype JobState
  (Idle)
  (Running Measurement3)
  (Failed i32))
```

### Traits

A public `deftrait` emits the trait name, documentation comment, and signature of each method.

```lisp
;;; Maps a value to a stable integer.
#[pub]
(deftrait StableHash [T]
  (stable-hash [(self : [& Self])] -> i32))
```

### Constants

```lisp
;;; Pi.
#[pub]
(const PI : f32 3.14159265f)
```

---

## Output Format

`gu-cli doc` generates a Markdown API reference. A file named after the package is created in the output directory.

```
doc/
└── {package-name}.md   # API reference containing functions, types, and constants
```

---

## Documentation Guidelines

Use the following guidelines to write effective documentation.

### Summarize on the First Line

The first line of a documentation comment should concisely summarize the definition.

```lisp
;;; Normalizes a vector.               <-- Summary
;;;
;;; Returns a vector of length one.     <-- Detailed description
;;; Behavior is undefined for a zero input vector.
#[pub]
(defn normalize [(v : Vec3)] -> Vec3 ...)
```

### Describe Arguments and Return Values

Document arguments and return values when they have special constraints or conditions.

```lisp
;;; Searches an array with binary search.
;;;
;;; The array must be sorted.
;;; Returns Ok(index) when the element is found,
;;; or Err(insert-position) when it is not found.
#[pub]
(defn binary-search [(arr : [Array i32 N]) (target : i32)] -> [Result i32 i32]
  ...)
```

### Include Examples

Examples make complex functions easier to understand.

```lisp
;;; Splits a string on a delimiter.
;;;
;;; Examples:
;;;   (String/split "a,b,c" ",") ;=> ["a", "b", "c"]
;;;   (String/split "hello" ",") ;=> ["hello"]
#[pub]
(defn String/split [(s : String) (delim : String)] -> [Vec String]
  ...)
```
