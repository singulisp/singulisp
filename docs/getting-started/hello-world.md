# Hello, World!

Learn how to write, compile, and run your first Singulisp program.

## Your first program

Create a file named `hello.gu` with the following contents:

```lisp
#[external]
(defn main [] -> ()
  (println! "Hello, Singulisp!"))
```

## Run it

The `gu-cli run` command compiles and runs the program in one step.

```bash
gu-cli run hello.gu
```

Output:

```
Hello, Singulisp!
```

## Build, then run

You can also compile and run the program separately.

```bash
gu-cli build hello.gu
./build/hello
```

`gu-cli build` produces a native binary in `./build/` by default. Use `-o` to specify another output path explicitly.

## Code walkthrough

This short program contains several fundamental Singulisp elements.

### S-expression syntax

Singulisp belongs to the Lisp family. Compound expressions are written as **S-expressions** of the
form `(operator argument ...)`. Atoms and literals are also expressions on their own.

### `defn` --- function definitions

`defn` defines a function.

```
(defn function-name [parameter-list]
  body)

(defn function-name [parameter-list] -> return-type
  body)
```

`main` is the special entry-point function of a program.

### The `#[external]` attribute

`#[external]` declares that a function may invoke external side effects such as I/O. Input and output
operations such as `println!` require the caller to be in an external context. `main` is implicitly
external when the attribute is omitted, but this example states the intent explicitly.

### The `println!` built-in form

`println!` is a reserved built-in output form that writes a string to standard output. The trailing
`!` is part of its name; an arbitrary name ending in `!` does not thereby become a user macro.

### The `-> ()` return type

`-> ()` means that the function returns no value—that is, it returns the Unit type.

## A program with a function

Here is a slightly more involved program that defines and calls a function.

```lisp
#[external]
(defn main [] -> ()
  (let x 42)
  (let y (double x))
  (println! "Result: {}" y))

(defn double [(n : i32)] -> i32
  (* n 2))
```

Running it produces:

```
Result: 84
```

### `let` --- variable bindings

`let` declares an immutable variable.

```lisp
(let x 42)         ;; The type of x is inferred from its use context
(let y (double x)) ;; Bind the function's return value
```

Because Singulisp supports type inference, type annotations can often be omitted.

### Parameter annotations

Parameter type annotations are optional. When necessary, write them explicitly as follows:

```lisp
(defn double [(n : i32)] -> i32
  (* n 2))
```

`(n : i32)` means that `n` has type `i32`. Parameter lists are enclosed in square brackets, `[...]`.

### Function calls

Function calls are S-expressions. `(double x)` calls `double` with the argument `x`.

Likewise, `(* n 2)` passes `n` and `2` to the multiplication operator `*`. Operators use the same call syntax as functions.

## Next steps

Once you understand the basics, continue to the [Language Quick Tour](quick-tour.md) for an overview of the major features.
See [Examples](../../examples/README.md) for more runnable programs.
