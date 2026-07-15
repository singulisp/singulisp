# Effect Call Boundaries

Singulisp uses attributes to classify external I/O and diagnostic processing, and statically rejects calls that are not permitted. Effect classification is not complete effect inference. Even functions without attributes may perform local mutation, allocation, and traps, so the classification does not guarantee referential transparency, such as always producing the same result for the same arguments.

## Three Function Categories

| Category | Designation | Primary use |
|---|---|---|
| Ordinary function | No attribute | Computation, local mutation, allocation |
| external | `#[external]` | Standard I/O, files, time, processes, FFI |
| diagnostics | `#[diagnostics]` | Instrumentation and diagnostics that can be removed by the build configuration |

An ordinary function cannot directly call an external or diagnostics function. An external context may invoke external side effects. Although the implementation treats `main` as external even when the attribute is omitted, public code should specify `#[external]` explicitly.

```lisp
(defn double [(value : i32)] -> i32
  (* value 2))

#[external]
(defn main [] -> ()
  (println! "result={}" (double 21)))
```

`format` is an ordinary operation that constructs a string value, so it does not require an external context. `println!`, `time/now-ms`, file APIs, and similar operations do require an external context.

## Call Rules

- Ordinary function → ordinary function: permitted
- Ordinary function → external / diagnostics: prohibited
- external → ordinary function / external / diagnostics: permitted
- diagnostics → ordinary function / external / diagnostics: permitted
- diagnostics gate → diagnostics: permitted in debug mode
- diagnostics gate → direct call to external: prohibited

Violations are checked at each call. The principal diagnostics are `E-EFF-0001` for a call from an ordinary function to an external function, `E-DIAG-0001` for an invalid call to a diagnostics function, and `E-DIAG-0002` for a direct call from a diagnostics gate to an external function.

`external` and `diagnostics` are attributes that determine call permissions, not mutually exclusive levels in a type hierarchy.

## diagnostics and Gates

To invoke diagnostic output from ordinary computation, separate the external processing from the conditional gate.

```lisp
#[diagnostics]
#[external]
(defn write-trace [(value : i32)] -> ()
  (println! "trace={}" value))

#[cfg-when (:mode :debug) :on-false :erase-call :kind :diagnostics]
(defn trace-value [(value : i32)] -> ()
  (write-trace value))

(defn process [(input : i32)] -> i32
  (let result (* input 2))
  (trace-value result)
  result)
```

A gate must have type `-> ()` and must be used in a position where the entire call can be removed when disabled. It cannot replace an expression that returns a value. In addition, arguments to a call removed by `:erase-call` are limited to atoms such as literals and variables. Bind complex expressions to a `let` first.

```lisp
;; Permitted
(let measured (+ a b))
(trace-value measured)

;; Prohibited: the erase-call argument is a compound expression
(trace-value (+ a b))
```

## Design Guidelines

- Put computation and data transformation in ordinary functions.
- Keep files, networking, time, standard I/O, and FFI at the external boundary.
- Split observations that should be removable in release builds into a diagnostics function and a diagnostics gate.
- Do not infer strict purity, including freedom from allocation and traps, from this classification alone.

See [Attributes](attributes.md) for conditional-attribute syntax and [Input and Output](../standard-library/io.md) for the I/O API.
