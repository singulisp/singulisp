# Code Formatter

`gu-cli fmt` reads `.gu` source through the same concrete syntax tree (CST) as the parser and applies deterministic formatting. Its input may be a single file or a project directory.

## Usage

```bash
# Overwrite a file
gu-cli fmt src/main.gu

# Overwrite every .gu file in the project
gu-cli fmt .

# Make no changes and report whether a diff exists through the exit status
gu-cli fmt src/main.gu --check

# Make no changes and write the formatted result to standard output
gu-cli fmt src/main.gu --stdout
```

Do not specify `--check` and `--stdout` together. The input is not modified if parsing fails.

## Basic Formatting

- Indentation is two spaces
- The target line length is 80 characters
- Top-level forms are separated by one blank line
- Trailing whitespace is removed, and nonempty content ends with one newline
- The order of `;`, `;;`, and `;;;` comments is preserved

Do not omit the colon in typed parameters and fields.

```lisp
(defstruct Point (x : f32) (y : f32))

(defn clamp-positive [(x : i32)] -> i32
  (if (> x 0)
    x
    0))
```

Write attribute arguments as keywords. Attributes may appear immediately before a definition or at the beginning of the same top-level form.

```lisp
#[inline :always]
(defn add [(a : i32) (b : i32)] -> i32
  (+ a b))
```

## CI

Specify the formatting target and use `--check`.

```yaml
- name: Check Singulisp formatting
  run: gu-cli fmt . --check
```

The CLI and LSP use the same formatter implementation, so format-on-save in the editor produces the same result as CI.
